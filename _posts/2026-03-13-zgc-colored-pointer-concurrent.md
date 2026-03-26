---
title: "ZGC — Colored Pointer 구조와 동시 수집으로 10ms 미만 pause 달성하기"
categories:
  - Dev Notes
tags:
  - GC
  - JVM
lang: ko
date: 2026-03-13
excerpt: "ZGC의 Colored Pointer 64비트 구조, Read Barrier 기반 Self-Healing, 수집 단계별 동작을 정리한다."
---

> **GC Deep Dive 시리즈**
> [1편. G1 GC — Region 기반 메모리와 수집 과정](/dev%20notes/g1-gc-region-memory-layout/)
> 2편. ZGC — Colored Pointer와 동시 수집 *(현재 글)*

---

## ZGC가 필요한 이유

G1 GC는 잘 만들어진 GC다. 하지만 힙이 수십 GB를 넘어가면 한계가 드러난다. Mixed GC의 STW가 수백 ms까지 올라가고, Humongous 객체 처리도 부담이 된다. 실시간성이 중요한 서비스(금융 거래, 게임 서버, 대규모 인메모리 캐시)에서는 이 정도 pause도 문제가 된다.

ZGC는 JDK 11에서 실험적으로 도입되고, JDK 15에서 Production Ready가 된 GC다. 목표가 명확하다: **힙 크기와 관계없이 pause time을 10ms 미만으로 유지한다.** 8MB부터 16TB 힙까지 지원한다.

어떻게 가능한지 하나씩 뜯어보자.

## ZGC Page Layout

![ZGC Colored Pointer & Concurrent Collection](/assets/images/jvm/zgc-architecture.png)

G1이 Region이라면 ZGC는 **Page**라는 단위를 쓴다. 세 가지 크기가 있다:

| Page 타입 | 크기 | 용도 |
|---|---|---|
| **Small** | 2MB | 256KB 이하 객체 |
| **Medium** | 32MB | 256KB ~ 4MB 객체 |
| **Large** | 가변 (2MB의 배수) | 4MB 초과 대형 객체 |

G1의 Humongous와 비슷하게 Large Page가 있지만, ZGC에서는 Large Page에 객체가 딱 하나만 들어간다. 그리고 Large Page는 **재배치(relocate) 대상에서 제외**된다. 이미 독립적인 공간이라 옮길 필요가 없기 때문이다.

G1과 결정적으로 다른 점은 **ZGC에는 세대(Generation) 구분이 없다**는 것이다. JDK 21에서 Generational ZGC가 도입되면서 이 부분이 바뀌긴 했는데, 기본 ZGC는 세대 없이 전체 힙을 대상으로 동작한다.

## Colored Pointer — 64비트 포인터에 메타데이터 심기

ZGC의 핵심 아이디어가 바로 **Colored Pointer**다. 객체를 가리키는 포인터 자체에 GC 메타데이터를 인코딩한다.

64비트 포인터의 구조:

```
|  18비트  | 1 | 1 | 1 | 1 |       42비트        |
| Unused  | F | R |M1 |M0 | Object Address     |
```

각 비트의 의미:

| 비트 | 이름 | 설명 |
|---|---|---|
| 63~46 | Unused | 사용하지 않음 (향후 확장용) |
| 45 | **Finalizable** | finalize()가 필요한 객체인지 여부 |
| 44 | **Remapped** | 최신 주소로 갱신 완료된 포인터인지 |
| 43 | **Marked1** | GC 마킹 비트 (홀수 사이클) |
| 42 | **Marked0** | GC 마킹 비트 (짝수 사이클) |
| 41~0 | **Object Address** | 실제 객체 주소 (4TB 주소 공간) |

42비트 주소면 최대 4TB다. 충분한가? 대부분의 경우 충분하고, JDK 버전이 올라가면서 주소 공간도 확장되고 있다.

Marked0과 Marked1을 번갈아 사용하는 이유가 있다. GC 사이클마다 마킹 비트를 초기화해야 하는데, 비트를 번갈아 쓰면 **이전 사이클의 마킹 정보를 지울 필요 없이 새 비트만 사용**하면 된다. 영리한 설계다.

### 왜 포인터에 색을 입히는가?

전통적인 GC는 객체 헤더(Mark Word)에 GC 상태를 기록한다. 이러면 객체에 접근해서 헤더를 읽어야 상태를 알 수 있다. ZGC는 **포인터만 보면 객체의 GC 상태를 즉시 알 수 있다.** 객체에 접근하지 않아도 된다.

이게 가능한 건 ZGC가 **멀티 매핑(Multi-Mapping)**을 사용하기 때문이다. 같은 물리 메모리를 여러 가상 주소에 매핑해서, 색상 비트가 달라도 같은 물리 메모리에 접근하게 만든다. Linux의 `mmap`으로 구현한다.

```
가상 주소 (Marked0 set):  0x0000_0400_0000_0000 + offset
가상 주소 (Marked1 set):  0x0000_0800_0000_0000 + offset
가상 주소 (Remapped set): 0x0000_1000_0000_0000 + offset
  ↓        ↓        ↓
  모두 같은 물리 메모리 페이지
```

## Read Barrier와 Self-Healing

G1은 **Write Barrier**를 사용한다. 참조를 쓸 때 RSet을 갱신하는 방식이다. ZGC는 반대로 **Read Barrier**를 사용한다. 참조를 읽을 때 포인터의 색상 비트를 확인한다.

```java
// 애플리케이션 코드
Object ref = obj.field;

// JIT가 삽입하는 Read Barrier (의사 코드)
Object ref = obj.field;
if (ref의 색상 비트가 현재 GC 상태와 불일치) {
    ref = slowPath(ref);  // 포인터를 최신 상태로 갱신
    obj.field = ref;       // ← Self-Healing: 원본도 갱신
}
```

여기서 **Self-Healing**이 핵심이다. 한 번 Read Barrier를 통과하면서 포인터가 갱신되면, **같은 참조를 다시 읽을 때는 slow path를 타지 않는다.** 원본 필드 자체가 갱신되었기 때문이다.

G1의 Write Barrier는 모든 참조 쓰기에 오버헤드가 있지만, ZGC의 Read Barrier는 Self-Healing 덕분에 **한 참조당 최대 한 번만 slow path가 발생**한다. 읽기가 쓰기보다 훨씬 빈번한 대부분의 워크로드에서, 이 trade-off가 의외로 나쁘지 않다.

## ZGC Collection Phases

ZGC의 수집 과정이다. STW 구간이 극도로 짧다는 걸 주목하자.

**Phase 1: Init Mark (STW < 1ms)**

GC Root를 스캔해서 Root에서 직접 참조하는 객체만 마킹한다. Root Set이 보통 작기 때문에 1ms 미만에 끝난다. 힙 크기와 무관하다.

**Phase 2: Concurrent Mark**

힙 전체를 탐색하면서 살아있는 객체를 마킹한다. 애플리케이션 스레드와 동시에 실행된다. Colored Pointer의 Marked0 또는 Marked1 비트를 설정한다.

**Phase 3: Remark (STW < 1ms)**

Concurrent Mark에서 미처 처리하지 못한 참조(SATB와 유사한 메커니즘)를 마무리한다. 역시 1ms 미만이다.

**Phase 4: Concurrent Prepare for Relocate**

수집할 Page를 선정한다. 살아있는 객체가 적은(가비지가 많은) Page를 골라 **Relocation Set**에 넣는다.

**Phase 5: Relocate Start (STW < 1ms)**

GC Root가 가리키는 객체 중 Relocation Set에 포함된 것들을 새 위치로 이동시킨다. Root만 처리하므로 역시 1ms 미만이다.

**Phase 6: Concurrent Relocate**

Relocation Set에 있는 나머지 객체들을 새 Page로 복사한다. 이동된 객체의 원래 주소와 새 주소를 **Forwarding Table**에 기록한다. 애플리케이션이 이동된 객체에 접근하면 Read Barrier가 Forwarding Table을 참조해서 새 주소로 리다이렉트한다.

**Phase 7: Concurrent Remap**

이전 사이클에서 이동된 객체의 참조를 새 주소로 갱신한다. 사실 이 작업은 다음 GC 사이클의 Concurrent Mark와 합쳐서 처리된다. 어차피 모든 참조를 순회하니까 그때 같이 하는 게 효율적이다.

```
전체 사이클 타임라인:

STW(~1ms) → Concurrent → STW(~1ms) → Concurrent → STW(~1ms) → Concurrent → Concurrent
Init Mark   Mark          Remark      Prepare       Relocate    Relocate     Remap
                                                    Start
```

STW가 총 3번 발생하고, 각각 1ms 미만이다. 나머지는 전부 concurrent다. 힙이 1TB든 10TB든 STW 시간은 거의 동일하다.

## G1 vs ZGC Trade-off

ZGC가 무조건 좋은 건 아니다.

| 비교 항목 | G1 | ZGC |
|---|---|---|
| pause time | 수십~수백 ms | < 10ms (보통 < 1ms) |
| throughput | 높음 | G1 대비 약간 낮음 (~5~15%) |
| 메모리 오버헤드 | RSet (힙의 5~10%) | Multi-Mapping + Forwarding Table |
| 세대 구분 | 있음 | 없음 (JDK 21부터 Generational ZGC) |
| 최소 JDK | JDK 9 (기본) | JDK 15 (Production Ready) |

**처리량(throughput)이 약간 희생**된다. Read Barrier 비용과, concurrent 작업에 CPU를 나눠 써야 하기 때문이다. 배치 처리처럼 latency보다 throughput이 중요한 워크로드에서는 G1이 나을 수 있다.

반대로 응답 시간 p99가 중요한 API 서버, 수십 GB 이상 힙을 쓰는 인메모리 캐시 서비스에서는 ZGC가 확실한 이점이 있다.

## 실전 설정

```bash
# ZGC 활성화
-XX:+UseZGC

# Generational ZGC (JDK 21+)
-XX:+UseZGC -XX:+ZGenerational

# 힙 크기 설정
-Xms4g -Xmx4g

# Concurrent GC 스레드 수 (기본: CPU 코어의 25%)
-XX:ConcGCThreads=4
```

ZGC는 튜닝 옵션이 G1보다 훨씬 적다. `-Xmx`만 적절히 잡으면 나머지는 JVM이 알아서 처리한다. 오히려 튜닝할 게 별로 없다는 게 장점이다.

## 마무리

ZGC의 설계 철학은 **"STW에서 할 일을 최소화하고, 나머지는 전부 concurrent로 처리한다"**는 것이다. Colored Pointer로 포인터 자체에 GC 상태를 인코딩하고, Read Barrier + Self-Healing으로 concurrent relocation을 가능하게 만들었다.

G1과 ZGC 중 뭘 쓸지는 결국 워크로드에 달려 있다. latency-sensitive한 서비스라면 ZGC를, throughput-first라면 G1을 선택하면 된다. JDK 21의 Generational ZGC가 throughput 격차를 많이 줄였으니, 앞으로는 ZGC의 비중이 더 커질 것 같다.

## Reference

- [Alibaba Cloud - JVM GC Deep Dive](https://www.alibabacloud.com/blog/601536)
