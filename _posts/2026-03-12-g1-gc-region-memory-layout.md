---
title: "G1 GC — Region 기반 메모리 구조와 수집 과정 정리"
categories:
  - Dev Notes
tags:
  - GC
  - JVM
lang: ko
date: 2026-03-30
excerpt: "G1 GC의 Region 분할 방식, RSet/Card Table 동작, Young GC와 Mixed GC 수집 과정을 정리한다."
---

> **GC Deep Dive 시리즈**
> 1편. G1 GC — Region 기반 메모리와 수집 과정 *(현재 글)*
> [2편. ZGC — Colored Pointer와 동시 수집](/dev%20notes/zgc-colored-pointer-concurrent/)

---

## 왜 G1 GC인가

JDK 9부터 기본 GC가 G1으로 바뀌었다. CMS가 deprecated 된 이후로 사실상 프로덕션에서 가장 많이 쓰이는 GC다. 그런데 막상 "Region이 뭔데?"라고 물으면 정확하게 설명하기 어려웠다. 이번에 제대로 정리해본다.

## Region 기반 메모리 구조

![G1 GC Region Based Memory Layout](/assets/images/jvm/g1-gc-architecture.png)

기존 GC(Serial, Parallel, CMS)는 힙을 Young(Eden + Survivor)과 Old로 물리적으로 나눈다. 연속된 메모리 블록이라 Old 영역이 크면 Full GC 시간이 길어질 수밖에 없다.

G1은 접근이 다르다. **힙 전체를 동일 크기의 Region으로 잘게 쪼갠다.** 기본적으로 1MB~32MB 사이인데, 힙 크기에 따라 JVM이 자동으로 결정한다. 보통 2048개 내외의 Region이 만들어진다.

각 Region은 다음 중 하나의 역할을 가진다:

| Region 타입 | 설명 |
|---|---|
| **Eden** | 새 객체가 할당되는 영역 |
| **Survivor** | Young GC에서 살아남은 객체가 복사되는 영역 |
| **Old** | 오래 살아남은 객체가 승격되는 영역 |
| **Humongous** | Region 크기의 50%를 초과하는 대형 객체 전용 |
| **Free** | 아직 어떤 역할도 할당되지 않은 빈 영역 |

핵심은 **역할이 고정이 아니라는 점**이다. 한 Region이 지금은 Eden이지만, GC 후에 Free가 되고, 나중에 Old가 될 수 있다. 이 유연성 때문에 G1이 pause time을 예측 가능하게 제어할 수 있다.

### Humongous Region 주의점

Region 크기의 50% 이상인 객체는 Humongous로 분류된다. 문제는 Humongous 객체가 연속된 여러 Region을 차지할 수 있다는 것이다. 예를 들어 Region이 4MB인데 10MB 객체가 들어오면 3개의 연속 Region을 사용한다.

Humongous 객체는 Young GC에서 수집되지 않고, Mixed GC나 Full GC에서만 회수된다. 그래서 큰 byte 배열이나 컬렉션을 무분별하게 만들면 Humongous 할당이 빈번해지고, GC 효율이 떨어진다.

```java
// Humongous 할당이 발생하는 전형적인 케이스
byte[] largeBuffer = new byte[4 * 1024 * 1024]; // 4MB — Region 크기에 따라 Humongous

// 이런 패턴은 피하자
List<byte[]> chunks = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    chunks.add(new byte[2 * 1024 * 1024]); // 반복적인 대형 할당
}
```

## Remember Set (RSet)과 Card Table

Region이 독립적으로 수집되려면 한 가지 문제를 풀어야 한다. **다른 Region에서 이 Region 안의 객체를 참조하고 있는지** 어떻게 알 것인가?

Young GC를 한다고 하자. Eden Region의 객체가 살아있는지 확인하려면 Old Region에서 참조하는지도 봐야 한다. 그렇다고 Old 전체를 스캔하면 Young GC의 의미가 없다.

이 문제를 해결하는 게 **RSet(Remembered Set)**이다.

각 Region은 자신만의 RSet을 가진다. RSet에는 **"나를 참조하는 다른 Region의 카드 정보"**가 기록되어 있다. 여기서 카드(Card)는 512바이트 단위의 메모리 블록이다.

동작 방식을 좀 더 구체적으로 보면:

1. Old Region의 객체 A가 Eden Region의 객체 B를 참조한다
2. 이 참조가 생기는 시점에 **Write Barrier**가 작동한다
3. A가 위치한 카드를 dirty로 마킹한다
4. 별도의 **Refinement Thread**가 dirty 카드를 처리해서 B가 속한 Region의 RSet에 기록한다

```
Old Region [Card 0][Card 1][Card 2 - dirty]...[Card N]
                              |
                              ↓
Eden Region의 RSet: "Old Region의 Card 2에서 나를 참조함"
```

이렇게 하면 Young GC 때 Old 전체를 스캔할 필요 없이, **RSet만 보면 외부 참조를 빠르게 파악**할 수 있다. 단점은 RSet 자체가 힙의 일부를 차지한다는 것이다. 보통 힙의 5~10% 정도를 RSet이 사용한다.

## G1 GC Collection 과정

G1의 수집은 크게 **Young GC**와 **Mixed GC** 두 가지로 나뉜다.

### Young GC

Eden Region이 가득 차면 발생한다. 모든 Eden과 Survivor Region이 대상이다.

1. STW(Stop-The-World) 발생
2. GC Root에서 시작해서 살아있는 객체를 탐색
3. 살아있는 객체를 새로운 Survivor Region으로 복사 (age가 임계치를 넘으면 Old로 승격)
4. 기존 Eden/Survivor Region은 Free로 반환

Young GC는 비교적 빠르다. Eden Region 수를 조절해서 pause time 목표(`-XX:MaxGCPauseMillis`, 기본 200ms)에 맞추려 한다.

### Mixed GC (Concurrent Marking + Mixed Collection)

Old 영역 사용량이 `InitiatingHeapOccupancyPercent`(기본 45%)를 넘으면 Concurrent Marking이 시작되고, 이후 Mixed GC가 수행된다.

**Phase 1: Initial Mark (STW)**

GC Root에서 직접 참조하는 객체만 마킹한다. Young GC에 편승(piggyback)해서 실행되기 때문에 별도의 STW가 추가되지 않는다.

**Phase 2: Root Region Scan**

Survivor Region을 스캔해서 Old Region으로의 참조를 찾는다. 애플리케이션 스레드와 동시에 실행된다. 다음 Young GC가 시작되기 전에 반드시 완료되어야 한다.

**Phase 3: Concurrent Marking**

힙 전체에서 살아있는 객체를 마킹한다. 애플리케이션과 동시에 실행된다. 이때 **SATB(Snapshot-At-The-Beginning)** 방식을 사용한다.

SATB는 마킹 시작 시점의 객체 그래프 스냅샷을 기준으로 "그 시점에 살아있던 객체는 모두 살아있다고 간주"하는 전략이다. Concurrent Marking 도중 애플리케이션이 참조를 변경하면, **pre-write barrier**가 변경 전 참조값을 SATB 큐에 기록한다.

```
// Concurrent Marking 중 참조 변경 시
obj.field = newRef;
// → pre-write barrier가 oldRef를 SATB 큐에 push
// → oldRef가 가리키던 객체는 이번 사이클에서 살아있다고 간주
```

이 방식의 장점은 마킹 도중 참조가 끊겨도 객체가 잘못 수집되는 일이 없다는 것이다. 대신 floating garbage(이미 죽었지만 이번에 못 거둔 객체)가 약간 발생할 수 있다. 다음 사이클에서 회수되니까 큰 문제는 아니다.

**Phase 4: Remark (STW)**

SATB 큐에 남아있는 참조들을 처리하고, 마킹을 최종 확정한다. 짧은 STW가 발생한다.

**Phase 5: Cleanup / Evacuation**

- 각 Region의 살아있는 객체 비율을 계산한다
- 가비지가 많은 Region을 우선적으로 수집 대상에 넣는다 (**Garbage First**의 이름 유래)
- 선택된 Region의 살아있는 객체를 다른 Region으로 복사(Evacuation)하고, 원래 Region은 Free로 반환한다

Mixed GC는 Young Region과 선택된 Old Region을 함께 수집한다. 한 번에 모든 Old Region을 처리하지 않고, pause time 목표에 맞춰 몇 개씩 나눠서 처리한다.

## Young GC vs Mixed GC 정리

| 구분 | Young GC | Mixed GC |
|---|---|---|
| 트리거 | Eden 영역 가득 참 | Old 사용률 > IHOP(45%) |
| 대상 | Eden + Survivor | Eden + Survivor + 선택된 Old |
| Concurrent Marking | 불필요 | 필수 (수집 대상 선정용) |
| pause time | 짧음 (수~수십 ms) | 상대적으로 김 (수십~수백 ms) |

## 튜닝 포인트

실무에서 자주 건드리는 옵션 몇 가지:

```bash
# pause time 목표 (기본 200ms)
-XX:MaxGCPauseMillis=100

# Concurrent Marking 시작 임계치 (기본 45%)
-XX:InitiatingHeapOccupancyPercent=35

# Mixed GC에서 한 번에 수집할 Old Region 최대 수 (기본 8)
-XX:G1MixedGCCountTarget=8

# Region 크기 직접 지정 (보통 자동에 맡김)
-XX:G1HeapRegionSize=4m
```

`MaxGCPauseMillis`를 너무 낮게 잡으면 Young Region 수가 줄어들어 Young GC가 자주 발생하고, Old 승격이 빨라져서 오히려 Mixed GC나 Full GC가 늘어날 수 있다. 적정 값을 찾는 게 중요하다.

## 마무리

G1 GC의 핵심은 결국 **"힙을 Region으로 쪼개서, 가비지가 많은 Region부터 선택적으로 수집한다"**는 것이다. RSet으로 Region 간 참조를 추적하고, SATB로 Concurrent Marking의 정확성을 보장한다. 완벽하지는 않지만(RSet 오버헤드, Humongous 문제 등), 대부분의 서버 워크로드에서 합리적인 성능을 보여준다.

다음 글에서는 G1의 한계를 넘어서려는 ZGC의 Colored Pointer와 sub-millisecond pause에 대해 정리할 예정이다.

## Reference

- [Alibaba Cloud - JVM GC Deep Dive](https://www.alibabacloud.com/blog/601536)
