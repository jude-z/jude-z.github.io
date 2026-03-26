---
title: "JVM 스레드 종류와 스택 프레임 내부 구조"
categories:
  - Dev Notes
tags:
  - JVM
  - Java
lang: ko
date: 2026-03-07
excerpt: "JVM이 내부적으로 관리하는 시스템 스레드와 Stack Frame의 Local Variables, Operand Stack, Frame Data를 정리한다."
---

> **JVM Internals 시리즈**
> [1편. JVM 아키텍처와 런타임 데이터 영역](/dev%20notes/jvm-architecture-runtime-data/)
> [2편. 클래스 로딩과 실행 엔진](/dev%20notes/jvm-classloader-execution-engine/)
> 3편. JVM 스레드와 스택 프레임 구조 *(현재 글)*

---

## JVM 안에는 생각보다 많은 스레드가 돌고 있다

`public static void main(String[] args)`으로 프로그램을 시작하면, 우리가 작성한 코드를 실행하는 main 스레드 하나만 있다고 생각하기 쉽다. 그런데 실제로 JVM을 띄우면 **백그라운드에서 여러 시스템 스레드**가 같이 돌아간다.

![JVM Threads & Stack Frame](/assets/images/jvm/jvm-thread-stack-frame.png)

간단하게 확인해볼 수 있다:

```java
public class ThreadList {
    public static void main(String[] args) {
        // 현재 실행 중인 모든 스레드 출력
        Thread.getAllStackTraces().keySet().forEach(t ->
            System.out.println(t.getName() + " [" + t.isDaemon() + "]")
        );
    }
}
```

출력하면 main 외에도 `Reference Handler`, `Finalizer`, `Signal Dispatcher` 같은 스레드가 보인다.

## JVM System Threads

HotSpot JVM이 내부적으로 관리하는 주요 시스템 스레드를 정리하면:

### VM Thread

JVM 내부 작업 중 **Safe Point**가 필요한 연산을 처리한다. Safe Point란 모든 스레드가 일시 중단되는 지점인데, GC의 Stop-the-World가 대표적이다. VM Thread가 이런 작업들을 실행하기 위해 다른 스레드가 Safe Point에 도달할 때까지 기다린다.

### GC Threads

Garbage Collection을 실제로 수행하는 스레드다. GC 알고리즘에 따라 하나 또는 여러 개의 GC 스레드가 동작한다.

```bash
# GC 스레드 수 확인/설정
java -XX:ParallelGCThreads=4 -jar app.jar
java -XX:ConcGCThreads=2 -jar app.jar
```

G1 GC나 ZGC는 **concurrent** GC 스레드를 사용해서, 애플리케이션 스레드와 동시에 GC 작업을 수행한다. 이게 STW 시간을 줄이는 핵심이다.

### Compiler Threads

JIT 컴파일을 담당한다. 바이트코드를 네이티브 코드로 컴파일하는 작업은 별도의 Compiler Thread에서 비동기적으로 수행된다. 컴파일이 끝나면 해당 메서드의 실행이 Interpreter에서 컴파일된 네이티브 코드로 전환된다.

### Signal Dispatcher Thread

OS로부터 전달되는 **시그널**을 받아서 적절한 JVM 내부 핸들러로 전달한다. `kill -3 <pid>`로 Thread dump를 뜨거나, `SIGTERM`으로 JVM을 종료할 때 이 스레드가 시그널을 처리한다.

### Periodic Task Thread

주기적인 타이머 이벤트(샘플링, JVM 내부 통계 수집 등)를 처리한다. JIT 컴파일 대상을 정할 때 사용하는 메서드 호출 빈도 측정도 이 스레드가 관여한다.

## Application Thread와 Stack

우리가 직접 사용하는 애플리케이션 스레드를 보자. 1편에서 정리했듯이, 각 스레드는 생성될 때 **자신만의 JVM Stack**을 할당받는다. 그리고 메서드가 호출될 때마다 Stack Frame이 push된다.

```java
public class OrderService {
    public void createOrder(Long userId) {       // Frame 1
        User user = userService.findById(userId); // Frame 2 push
        Order order = new Order(user);
        paymentService.process(order);             // Frame 3 push
    }
}
```

이 호출 과정에서 스택은 이렇게 변한다:

```
[createOrder] → [createOrder][findById] → [createOrder] → [createOrder][process] → [createOrder]
```

스레드 덤프를 찍으면 이 스택의 현재 상태를 볼 수 있다:

```bash
# Thread dump 생성
jstack <pid>

# 또는 코드에서
Thread.currentThread().getStackTrace();
```

## Stack Frame 내부 구조

Stack Frame이 그냥 "메서드 호출 정보를 저장하는 곳" 정도로만 알고 있었는데, 내부를 들여다보면 세 가지 영역으로 나뉜다.

### Local Variables Array

**지역 변수와 메서드 파라미터**를 저장하는 배열이다. 인덱스로 접근하며, 0번 인덱스는 인스턴스 메서드의 경우 `this` 참조가 들어간다.

```java
public int calculate(int a, int b) {
    int result = a + b;    // result는 index 3
    return result;
}

// Local Variables Array:
// [0] this      (인스턴스 메서드이므로)
// [1] a         (파라미터)
// [2] b         (파라미터)
// [3] result    (지역 변수)
```

static 메서드는 `this`가 없으므로 0번부터 파라미터가 시작된다:

```java
public static int add(int a, int b) {
    return a + b;
}

// Local Variables Array:
// [0] a
// [1] b
```

`long`이나 `double` 같은 64비트 타입은 슬롯 2개를 차지한다:

```java
public void example(long x, int y) {
    // [0] this
    // [1-2] x  (long → 슬롯 2개)
    // [3] y
}
```

실제 바이트코드에서 확인할 수 있다:

```bash
# 바이트코드 확인
javap -v MyClass.class
```

### Operand Stack

바이트코드 명령어가 **연산을 수행할 때 사용하는 작업 스택**이다. 산술 연산, 메서드 호출의 인자 전달, 반환 값 수신 등이 이 스택을 통해 이루어진다.

간단한 덧셈 `int c = a + b;`가 바이트코드 수준에서 어떻게 동작하는지 보자:

```
iload_1        // Local Variables[1] (a)을 Operand Stack에 push
               // Stack: [a]

iload_2        // Local Variables[2] (b)을 Operand Stack에 push
               // Stack: [a, b]

iadd           // Stack에서 두 값을 pop, 더한 결과를 push
               // Stack: [a+b]

istore_3       // Stack에서 pop한 값을 Local Variables[3] (c)에 저장
               // Stack: []
```

Operand Stack의 최대 깊이는 컴파일 타임에 결정되며, `.class` 파일에 `max_stack`으로 기록된다.

좀 더 복잡한 예제를 보면:

```java
public int complex(int x) {
    return (x + 1) * (x - 1);
}
```

```
iload_1        // Stack: [x]
iconst_1       // Stack: [x, 1]
iadd           // Stack: [x+1]
iload_1        // Stack: [x+1, x]
iconst_1       // Stack: [x+1, x, 1]
isub           // Stack: [x+1, x-1]
imul           // Stack: [(x+1)*(x-1)]
ireturn        // 결과 반환
```

이게 JVM이 **스택 기반 아키텍처**라고 불리는 이유다. 레지스터 기반(Android의 Dalvik/ART)과 대비되는 특성이다. 스택 기반의 장점은 하드웨어 레지스터 수에 의존하지 않아서 **이식성이 높다**는 것이다.

### Frame Data

Stack Frame의 세 번째 구성요소로, 다음을 포함한다:

**Runtime Constant Pool 참조**
- 현재 메서드가 속한 클래스의 Constant Pool을 가리키는 참조
- 바이트코드에서 상수나 메서드 참조가 필요할 때 이 참조를 통해 접근한다

```java
// 바이트코드에서 "Hello"나 System.out.println 같은 참조는
// Constant Pool을 통해 해석된다
System.out.println("Hello");
```

**예외 테이블 (Exception Table)**
- try-catch 블록에 대한 정보
- 예외가 발생했을 때 어떤 catch 블록으로 점프할지 결정한다

```java
public void riskyMethod() {
    try {
        // 바이트코드 범위: 0 ~ 10
        doSomething();
    } catch (IOException e) {
        // 핸들러 시작: 11
        handleError(e);
    }
}

// Exception Table:
// from=0, to=10, target=11, type=IOException
```

예외가 발생하면 JVM은 현재 Stack Frame의 예외 테이블을 검색한다. 매칭되는 핸들러가 없으면 해당 프레임을 pop하고 호출자의 예외 테이블을 검색한다. 이게 **예외 전파(Exception Propagation)** 메커니즘이다.

## Runtime Constant Pool

Method Area에 클래스별로 저장되는 영역으로, 컴파일 타임의 Constant Pool을 런타임으로 확장한 것이다. 숫자 리터럴, 문자열 리터럴, 필드 참조, 메서드 참조 등을 포함한다.

```java
public class Example {
    private static final String PREFIX = "user_";  // Constant Pool에 저장

    public String createKey(String id) {
        return PREFIX + id;  // PREFIX 참조는 Constant Pool을 통해 해석
    }
}
```

Stack Frame의 Frame Data가 이 Runtime Constant Pool에 대한 참조를 가지고 있어서, 바이트코드 실행 중에 필요한 상수나 심볼릭 참조를 빠르게 조회할 수 있다.

## 정리

3편에 걸쳐 JVM 내부 구조를 정리했다. 핵심만 요약하면:

- **Runtime Data Areas**: Heap과 Method Area는 공유, Stack/PC Register/Native Method Stack은 스레드별 독립
- **Class Loading**: Loading → Linking(Verification/Preparation/Resolution) → Initialization, Parent Delegation으로 핵심 클래스 보호
- **Execution Engine**: Interpreter로 시작, Hot Code는 JIT(C1→C2)로 컴파일
- **Stack Frame**: Local Variables Array + Operand Stack + Frame Data로 구성, 스택 기반 연산 수행

운영 환경에서 Thread dump를 분석하거나, GC 튜닝을 하거나, `StackOverflowError`를 디버깅할 때 이 구조를 알고 있으면 문제의 원인을 훨씬 빠르게 좁힐 수 있다.

## 참고

- [JVM Internals — James D Bloom](https://blog.jamesdbloom.com/JVMInternals.html)
- [The Java Virtual Machine Specification (Java SE 17)](https://docs.oracle.com/javase/specs/jvms/se17/html/)
