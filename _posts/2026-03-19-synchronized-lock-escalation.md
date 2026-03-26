---
title: "synchronized 락 에스컬레이션과 Object Header 깊이 파기"
categories:
  - Dev Notes
tags:
  - Concurrency
  - JVM
lang: ko
date: 2026-03-19
excerpt: "Java의 synchronized가 내부적으로 Biased → Lightweight → Heavyweight로 락을 업그레이드하는 과정과 Object Header 구조를 정리한다."
---

> **Java 동시성 시리즈**
> 1편. synchronized 락 에스컬레이션과 Object Header *(현재 글)*
> [2편. ReentrantLock, AQS, 그리고 동시성 제어](/dev%20notes/reentrantlock-aqs-concurrency/)

---

## 왜 이걸 공부하게 됐나

`synchronized`를 쓰면 무조건 무거운 락이 걸린다고 생각했다. 근데 JDK 6 이후로 JVM이 내부적으로 락 최적화를 꽤 많이 해놓았다는 걸 알게 됐다. 면접에서도 자주 나오는 주제이기도 하고, 실제로 Object Header의 Mark Word가 어떻게 변하면서 락이 에스컬레이션되는지를 제대로 이해하고 싶어서 정리하기 시작했다.

![synchronized Lock Escalation & Object Header](/assets/images/jvm/lock-escalation.png)

---

## Java Object Header 구조

Java에서 모든 객체는 힙에 저장될 때 Object Header를 갖는다. 이 헤더가 락의 핵심이다.

### Object Header 구성 (64-bit JVM 기준)

| 영역 | 크기 | 설명 |
|------|------|------|
| **Mark Word** | 8 bytes (64 bits) | 해시코드, GC 정보, 락 상태 |
| **Class Pointer (Klass Pointer)** | 4 bytes (압축 시) / 8 bytes | 어떤 클래스의 인스턴스인지 |
| **Instance Data** | 가변 | 실제 필드 데이터 |
| **Padding** | 0~7 bytes | 8바이트 정렬을 위한 패딩 |

여기서 핵심은 **Mark Word**다. 락 상태에 따라 Mark Word의 레이아웃이 완전히 바뀐다.

### Mark Word 레이아웃 (32-bit 기준으로 설명)

```
|-----------------------------------------------|-------|
|             Mark Word (32 bits)                | State |
|-----------------------------------------------|-------|
| HashCode(25) | GC Age(4) | Biased(1) | Lock(2)|       |
|-----------------------------------------------|-------|
| 무잠금 상태:  identity_hashcode | age | 0 | 01  | 무잠금 |
| 편향 잠금:    Thread ID | epoch | age | 1 | 01  | 편향   |
| 경량 잠금:    Lock Record 포인터        | 00    | 경량   |
| 중량 잠금:    Monitor 포인터            | 10    | 중량   |
| GC 마킹:                                | 11    | GC    |
|-----------------------------------------------|-------|
```

맨 뒤 2비트가 Lock Flag인데, 이 값에 따라 앞쪽 비트의 해석 방식 자체가 달라진다. 처음엔 이게 좀 헷갈렸는데, union 같은 느낌이라고 생각하면 된다.

---

## Lock Escalation 과정

`synchronized`에 진입하면 JVM은 **가장 가벼운 락부터 시도**하고, 경합이 심해질수록 무거운 락으로 올라간다.

### 1단계: Biased Lock (편향 잠금) — Lock Flag `01`, Biased `1`

가장 먼저 시도되는 락이다. **경합이 전혀 없는 상황**을 위한 최적화.

- 처음 락을 획득한 스레드의 **Thread ID를 Mark Word에 CAS로 기록**
- 같은 스레드가 다시 진입하면 CAS 없이 그냥 통과 (Thread ID만 비교)
- 다른 스레드가 접근하면 편향이 **철회(revoke)** 되면서 경량 잠금으로 업그레이드

실질적으로 **단일 스레드 환경에서 synchronized의 비용을 거의 0에 가깝게** 만드는 전략이다. 다만 JDK 15부터 기본 비활성화됐다(`-XX:-UseBiasedLocking`). 최신 JVM에서는 경량 잠금 최적화가 충분히 빠르기 때문이라고.

### 2단계: Lightweight Lock (경량 잠금) — Lock Flag `00`

두 개 이상의 스레드가 번갈아가며 접근하지만, **동시에 경합하진 않는** 상황에 적합하다.

동작 방식:

1. 현재 스레드의 **스택 프레임에 Lock Record 공간**을 할당
2. Mark Word의 내용을 Lock Record에 복사 (Displaced Mark Word)
3. **CAS로 Mark Word를 Lock Record 포인터로 교체** 시도
4. 성공하면 락 획득, 실패하면 스핀 후 중량 잠금으로 업그레이드

```
[ Thread Stack ]          [ Object Header ]
+---------------+         +------------------+
| Lock Record   | <------ | Mark Word: 00    |
| - Displaced   |         | (Lock Record ptr)|
|   Mark Word   |         +------------------+
+---------------+
```

여기서 주의할 점은, CAS가 실패한다고 바로 중량 잠금으로 올라가는 건 아니다. JVM이 **적응적 스핀(Adaptive Spinning)**을 먼저 시도한다. 이전에 스핀으로 성공한 이력이 있으면 더 오래 돌리고, 계속 실패하면 스핀 횟수를 줄인다.

### 3단계: Heavyweight Lock (중량 잠금) — Lock Flag `10`

**실제로 스레드 경합이 발생**할 때 진입하는 단계다. OS 레벨의 Mutex와 연관되기 때문에 비용이 크다.

- Mark Word가 **ObjectMonitor 포인터**를 가리킴
- 락을 못 얻은 스레드는 **커널 모드로 전환**되어 블로킹
- 컨텍스트 스위칭 비용이 발생

정리하면 이렇다:

| 단계 | Lock Flag | 조건 | 비용 |
|------|-----------|------|------|
| Biased | `01` (Biased=1) | 단일 스레드 | 거의 없음 |
| Lightweight | `00` | 번갈아 접근, 경합 없음 | CAS + 스핀 |
| Heavyweight | `10` | 실제 경합 | OS Mutex, 컨텍스트 스위칭 |

---

## Monitor (ObjectMonitor) 구조

중량 잠금 상태에서 사용되는 **ObjectMonitor**의 내부 구조를 보면 `synchronized`의 동작이 명확해진다.

```cpp
// HotSpot ObjectMonitor 주요 필드
ObjectMonitor {
    _owner        // 현재 락을 소유한 스레드
    _EntryList    // 락 획득을 대기하는 스레드 큐 (blocked)
    _WaitSet      // wait()을 호출하여 대기 중인 스레드 큐
    _count        // 진입 대기 중인 스레드 수
    _recursions   // 재진입 횟수
}
```

### 동작 흐름

1. 스레드가 `synchronized` 블록에 진입하면 `_owner`를 자신으로 설정
2. 이미 다른 스레드가 `_owner`면 → **_EntryList에 들어가서 블로킹**
3. `_owner`가 `wait()`을 호출하면 → **_WaitSet으로 이동**, `_owner` 해제
4. 다른 스레드가 `notify()`를 호출하면 → _WaitSet에서 하나를 꺼내 **_EntryList로 이동**
5. `notifyAll()`이면 → _WaitSet 전체를 _EntryList로 이동

```
              synchronized 진입 시도
                     |
              +------v------+
              |  _owner 확인 |
              +------+------+
                     |
           +----yes--+--no----+
           |                  |
      락 획득 성공        _EntryList에서
      _owner = self       대기 (blocked)
           |                  |
       wait() 호출시      _owner 해제 후
       _WaitSet 이동      재경쟁
```

---

## 재진입(Reentrancy) 메커니즘

`synchronized`는 **재진입이 가능**하다. 같은 스레드가 이미 락을 잡고 있는 객체의 `synchronized` 블록에 다시 진입할 수 있다.

```java
public class ReentrantExample {
    public synchronized void methodA() {
        System.out.println("methodA");
        methodB(); // 같은 락에 재진입 — 데드락 안 걸림
    }

    public synchronized void methodB() {
        System.out.println("methodB");
    }
}
```

이게 가능한 이유는 ObjectMonitor의 `_recursions` 필드 때문이다:

- 같은 스레드가 진입할 때마다 `_recursions++`
- 빠져나올 때마다 `_recursions--`
- `_recursions == 0`이 되면 `_owner`를 해제하고 락을 반납

경량 잠금 단계에서도 재진입이 가능한데, 이 경우 **스택에 Lock Record를 추가로 하나 더 쌓는** 방식으로 처리한다. Displaced Mark Word가 null인 Lock Record가 추가되고, 이걸 보고 재진입인지 판단한다.

---

## 정리

`synchronized`는 겉보기엔 단순한 키워드지만, 내부적으로는 JVM이 상당히 정교한 최적화를 하고 있다:

1. **Object Header의 Mark Word** 레이아웃이 락 상태에 따라 동적으로 변한다
2. **Biased → Lightweight → Heavyweight**로 점진적 에스컬레이션
3. 중량 잠금에서는 **ObjectMonitor**가 EntryList, WaitSet으로 스레드를 관리
4. **재진입**은 `_recursions` 카운터 또는 스택 Lock Record 추가로 처리

다음 편에서는 `synchronized`의 한계를 넘어서는 `ReentrantLock`과 그 기반인 AQS에 대해 정리한다.

---

**참고 자료:**
- [Let's Talk About Several of the JVM-Level Locks in Java - Alibaba Cloud](https://www.alibabacloud.com/blog/lets-talk-about-several-of-the-jvm-level-locks-in-java_596090)
