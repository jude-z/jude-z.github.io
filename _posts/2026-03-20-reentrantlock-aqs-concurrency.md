---
title: "ReentrantLock, AQS 내부 구조와 Java 동시성 제어 총정리"
categories:
  - Dev Notes
tags:
  - Concurrency
  - JVM
lang: ko
date: 2026-03-20
excerpt: "AQS의 CLH Queue와 state 필드부터 ReentrantLock, LongAdder, ReadWriteLock까지 Java 동시성 제어 메커니즘을 깊이 정리한다."
---

> **Java 동시성 시리즈**
> [1편. synchronized 락 에스컬레이션과 Object Header](/dev%20notes/synchronized-lock-escalation/)
> 2편. ReentrantLock, AQS, 그리고 동시성 제어 *(현재 글)*

---

## 들어가며

이전 편에서 `synchronized`의 내부 동작을 다뤘는데, 실무에서는 `ReentrantLock`을 쓰는 경우도 많다. 타임아웃이 필요하거나, 공정성을 제어하거나, Condition으로 세밀하게 스레드를 관리해야 할 때 `synchronized`만으로는 부족하다. 이번에는 `ReentrantLock`의 기반인 **AQS**를 중심으로 Java 동시성 제어를 전체적으로 정리해봤다.

![ReentrantLock, AQS & Concurrency Control](/assets/images/jvm/reentrantlock-aqs.png)

---

## synchronized vs ReentrantLock 비교

먼저 두 방식의 차이부터 정리하자. 면접에서도 단골 질문이다.

| 항목 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 잠금/해제 | 자동 (블록 종료 시) | 수동 (`lock()` / `unlock()`) |
| 공정성 | 불공정 (non-fair) 고정 | 공정/불공정 선택 가능 |
| 인터럽트 | 불가 | `lockInterruptibly()` |
| 타임아웃 | 불가 | `tryLock(timeout, unit)` |
| Condition | `wait()`/`notify()` 단일 | 여러 Condition 생성 가능 |
| 구현 레벨 | JVM 네이티브 (monitorenter/monitorexit) | Java 코드 (AQS 기반) |

`synchronized`는 편하지만, 락을 잡고 있는 스레드를 인터럽트할 수 없고, 타임아웃도 못 건다. `ReentrantLock`은 이런 부분을 해결하지만 **반드시 `finally`에서 `unlock()`을 호출**해야 한다. 안 그러면 데드락 직행이다.

```java
ReentrantLock lock = new ReentrantLock(true); // fair lock

lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // 반드시 finally에서
}
```

---

## AQS (AbstractQueuedSynchronizer) 내부 구조

`ReentrantLock`, `Semaphore`, `CountDownLatch`, `ReentrantReadWriteLock` — 이것들의 기반이 전부 **AQS**다. Doug Lea가 설계한 동기화 프레임워크인데, 핵심은 두 가지다.

### 1. state (volatile int)

```java
private volatile int state;
```

- 이 하나의 `int` 값으로 동기화 상태를 표현
- `ReentrantLock`: 0이면 미잠금, 1 이상이면 잠금 (재진입 횟수)
- `Semaphore`: 남은 permit 수
- `CountDownLatch`: 남은 카운트
- CAS(`compareAndSetState`)로 원자적 변경

### 2. CLH Queue (변형된 Craig-Landin-Hagersten Queue)

락을 못 얻은 스레드들이 대기하는 **이중 연결 리스트(doubly-linked list)**.

```
  head                                          tail
   |                                             |
   v                                             v
+------+    prev    +------+    prev    +------+
| Node | <--------- | Node | <--------- | Node |
| (set)|  next      | (T1) |  next      | (T2) |
|      | ---------> |      | ---------> |      |
+------+            +------+            +------+
  sentinel           WAITING             WAITING
```

- **head**는 센티넬 노드 (이미 락을 획득한 스레드 또는 더미)
- 새 스레드가 들어오면 **tail에 CAS로 노드를 추가**
- head의 다음 노드가 락 획득 시도 → 성공하면 자신이 새 head가 됨
- 각 노드는 `waitStatus` 필드를 가짐: `SIGNAL(-1)`, `CANCELLED(1)`, `CONDITION(-2)` 등

### Exclusive vs Shared 모드

AQS는 두 가지 모드를 지원한다:

**Exclusive (독점 모드)**
- 한 번에 하나의 스레드만 상태 획득
- `ReentrantLock`이 이걸 씀
- `tryAcquire()` / `tryRelease()` 오버라이드

**Shared (공유 모드)**
- 여러 스레드가 동시에 상태 획득 가능
- `Semaphore`, `CountDownLatch`, `ReadWriteLock`의 읽기 락
- `tryAcquireShared()` / `tryReleaseShared()` 오버라이드

```java
// ReentrantLock 내부의 tryAcquire (비공정 버전, 단순화)
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 아무도 안 잡고 있으면 CAS로 획득 시도
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        // 재진입: state 증가
        int nextc = c + acquires;
        setState(nextc);
        return true;
    }
    return false;
}
```

공정 락(`FairSync`)은 여기서 `hasQueuedPredecessors()` 체크가 추가된다. CLH Queue에 먼저 대기하고 있는 스레드가 있으면 양보하는 방식이다. 처리량은 떨어지지만 기아(starvation)를 방지한다.

---

## ReentrantReadWriteLock — state 분할 전략

읽기/쓰기 락을 분리해서 읽기 동시성을 높이는 구조다. 흥미로운 점은 **AQS의 state 하나로 읽기와 쓰기 상태를 동시에 관리**한다는 것이다.

```
state (32 bits)
+----------------+----------------+
|  상위 16 bits   |  하위 16 bits   |
|   읽기 잠금 수   |   쓰기 잠금 수   |
+----------------+----------------+
```

```java
// 읽기 잠금 수 추출
static int sharedCount(int c)    { return c >>> 16; }
// 쓰기 잠금 수 추출
static int exclusiveCount(int c) { return c & ((1 << 16) - 1); }
```

- 쓰기 잠금: `state`의 하위 16비트. exclusive 모드로 동작
- 읽기 잠금: `state`의 상위 16비트. shared 모드로 동작
- 쓰기 락 보유 중엔 다른 스레드의 읽기/쓰기 모두 차단
- 읽기 락만 보유 중엔 다른 스레드의 읽기는 허용, 쓰기는 차단

실무에서 **캐시 구현**할 때 유용하다. 읽기가 압도적으로 많은 상황에서 `synchronized`를 쓰면 읽기끼리도 블로킹되지만, `ReadWriteLock`을 쓰면 읽기는 동시에 가능하니까.

---

## LongAdder의 Segment 전략

`AtomicLong`은 단일 `volatile long`에 CAS를 걸기 때문에, 스레드가 많아지면 CAS 실패 → 재시도가 반복되면서 성능이 떨어진다. `LongAdder`는 이걸 분산시킨다.

```
LongAdder 내부 구조:
+------+
| base |  ← 경합 없을 때 여기에 CAS
+------+
+--------+--------+--------+--------+
| Cell[0]| Cell[1]| Cell[2]| Cell[3]|  ← 경합 시 스레드별 분산
+--------+--------+--------+--------+

sum() = base + Cell[0] + Cell[1] + ... + Cell[n]
```

- **경합이 없으면** `base`에 직접 CAS
- **CAS 실패(경합 감지)** 시 → `Cell` 배열을 생성하고 스레드를 특정 Cell에 매핑
- 매핑은 `Thread.getProbe()` 값의 해시로 결정 (threadLocalRandomProbe)
- 해당 Cell에서도 CAS 실패하면 → Cell을 rehash하거나 배열 크기를 확장
- `sum()` 호출 시 base + 모든 Cell의 합산

핵심은 **쓰기를 분산시키고, 읽기 시점에 합산**한다는 전략이다. 정확한 실시간 값이 필요한 게 아니라 카운터 용도라면 `AtomicLong` 대신 `LongAdder`를 쓰는 게 맞다.

---

## JVM 락 최적화 기법들

JVM은 `synchronized` 성능을 올리기 위해 런타임에 여러 최적화를 한다. 이전 편에서 다룬 Lock Escalation 외에도:

### Lock Elimination (락 제거)

JIT 컴파일러가 **Escape Analysis**를 통해 객체가 스레드를 벗어나지 않는다고 판단하면, `synchronized` 자체를 제거해 버린다.

```java
public String concat(String a, String b) {
    // StringBuffer는 synchronized 메서드를 갖고 있지만
    // sb가 이 메서드 밖으로 나가지 않으므로 → 락 제거
    StringBuffer sb = new StringBuffer();
    sb.append(a);
    sb.append(b);
    return sb.toString();
}
```

### Lock Coarsening (락 거칠게 만들기)

연속적으로 같은 객체에 락을 잡았다 풀었다 반복하면, JVM이 이를 **하나의 큰 락 범위로 합쳐**버린다.

```java
// 최적화 전: 3번의 lock/unlock
sb.append("a"); // lock → unlock
sb.append("b"); // lock → unlock
sb.append("c"); // lock → unlock

// JVM 최적화 후: 1번의 lock/unlock
lock();
sb.append("a");
sb.append("b");
sb.append("c");
unlock();
```

### Adaptive Spinning (적응적 스핀)

경량 잠금에서 CAS 실패 시 바로 블로킹하지 않고 스핀하는 것. **이전 스핀 성공/실패 이력**을 바탕으로 스핀 횟수를 동적으로 조절한다. 최근에 같은 락에서 스핀으로 성공한 적이 있으면 더 오래 스핀하고, 계속 실패하면 스핀을 아예 생략할 수도 있다.

---

## 정리

| 도구 | 핵심 메커니즘 | 적합한 상황 |
|------|-------------|-----------|
| `synchronized` | Object Header + Monitor | 간단한 동기화, 코드 간결성 |
| `ReentrantLock` | AQS + CLH Queue | 타임아웃, 공정성, Condition 필요 |
| `ReadWriteLock` | AQS state 분할 (16/16) | 읽기 >> 쓰기인 경우 |
| `LongAdder` | base + Cell 분산 | 고경합 카운터 |

결국 동시성 제어는 **경합의 정도와 패턴**에 따라 적절한 도구를 고르는 게 핵심이다. 경합이 없으면 Biased Lock이나 CAS 한 방이면 충분하고, 경합이 심하면 LongAdder처럼 아예 쓰기를 분산시키는 전략이 필요하다. 무조건 `synchronized`가 나쁘다거나 `ReentrantLock`이 좋다는 게 아니라, 상황에 맞게 쓰면 된다.

---

**참고 자료:**
- [Let's Talk About Several of the JVM-Level Locks in Java - Alibaba Cloud](https://www.alibabacloud.com/blog/lets-talk-about-several-of-the-jvm-level-locks-in-java_596090)
