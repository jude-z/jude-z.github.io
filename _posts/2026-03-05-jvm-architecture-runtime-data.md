---
title: "JVM 아키텍처 개요와 런타임 데이터 영역 정리"
categories:
  - Dev Notes
tags:
  - JVM
  - Java
lang: ko
date: 2026-03-05
excerpt: "JVM의 전체 구조를 잡고, Runtime Data Areas 각 영역이 어떤 역할을 하는지 정리한다."
---

> **JVM Internals 시리즈**
> 1편. JVM 아키텍처와 런타임 데이터 영역 *(현재 글)*
> [2편. 클래스 로딩과 실행 엔진](/dev%20notes/jvm-classloader-execution-engine/)
> [3편. JVM 스레드와 스택 프레임 구조](/dev%20notes/jvm-thread-stack-frame/)

---

## 왜 JVM 내부를 알아야 하는가

솔직히 말하면, 대부분의 Java 개발자가 JVM 내부를 몰라도 코드를 짤 수 있다. Spring Boot 띄우고 API 만들고 배포하는 데 Heap이 몇 개의 Generation으로 나뉘는지 알 필요는 없다.

그런데 **운영 환경에서 문제가 터지면** 얘기가 달라진다. OOM이 났는데 Heap dump를 보고도 뭐가 문제인지 모르겠다거나, GC 로그에 Full GC가 반복되는데 왜 그런지 감이 안 잡히거나. 이런 상황에서 JVM 구조를 아는 것과 모르는 것의 차이는 꽤 크다.

이 시리즈에서는 JVM 내부 구조를 3편에 걸쳐 정리한다. 첫 번째 글에서는 JVM의 전체 아키텍처와 Runtime Data Areas를 다룬다.

## JVM 전체 아키텍처

![JVM Architecture](/assets/images/jvm/jvm-architecture-overview.png)

JVM은 크게 세 파트로 나눌 수 있다:

1. **Class Loader Subsystem** — `.class` 파일을 읽어서 메모리에 올린다
2. **Runtime Data Areas** — 프로그램 실행에 필요한 데이터를 저장하는 메모리 공간
3. **Execution Engine** — 바이트코드를 실제로 실행한다

Class Loader와 Execution Engine은 다음 편에서 깊게 다룰 거고, 이 글에서는 Runtime Data Areas에 집중한다.

## Runtime Data Areas

JVM이 프로그램을 실행하면서 사용하는 메모리 영역이다. 중요한 건 이 영역들이 **스레드 공유 여부**에 따라 나뉜다는 점이다.

| 영역 | 공유 범위 | 생성 시점 |
|------|----------|----------|
| Heap | 모든 스레드 공유 | JVM 시작 시 |
| Method Area | 모든 스레드 공유 | JVM 시작 시 |
| JVM Stack | 스레드별 독립 | 스레드 생성 시 |
| PC Register | 스레드별 독립 | 스레드 생성 시 |
| Native Method Stack | 스레드별 독립 | 스레드 생성 시 |

이 구분이 중요한 이유는 **동시성 문제**와 직결되기 때문이다. 공유 영역에 있는 데이터는 여러 스레드가 동시에 접근할 수 있으니, synchronized나 volatile 같은 동기화 메커니즘이 필요해진다.

### Heap

Java 개발자가 가장 많이 듣는 영역. `new` 키워드로 생성한 **모든 객체 인스턴스와 배열**이 여기에 저장된다.

```java
// 이 객체는 Heap에 할당된다
User user = new User("jude", 28);

// 배열도 Heap
int[] scores = new int[100];

// String 리터럴은 String Pool(Heap 내부)에 저장
String name = "hello";
```

Heap은 GC(Garbage Collector)가 관리하는 영역이다. 더 이상 참조되지 않는 객체는 GC가 정리한다. JVM 옵션으로 크기를 조절할 수 있다:

```bash
# 초기 Heap 크기 256MB, 최대 1GB
java -Xms256m -Xmx1g -jar app.jar
```

Heap이 꽉 차면 `OutOfMemoryError: Java heap space`가 발생한다. 운영 환경에서 이 에러를 보면 보통 메모리 누수를 의심해야 한다.

### Method Area (Metaspace)

클래스 수준의 정보가 저장되는 곳이다:

- 클래스의 **런타임 상수 풀** (Runtime Constant Pool)
- 필드, 메서드 데이터
- 메서드와 생성자의 **바이트코드**
- static 변수

```java
public class UserService {
    // static 변수 → Method Area에 저장
    private static final int MAX_RETRY = 3;

    // 메서드의 바이트코드 → Method Area에 저장
    public User findById(Long id) {
        // ...
    }
}
```

Java 8부터는 기존의 PermGen이 **Metaspace**로 대체됐다. 가장 큰 차이는 Metaspace가 **Native Memory**를 사용한다는 것이다. PermGen은 Heap 안에 있어서 크기가 고정이었는데, Metaspace는 OS 메모리를 직접 사용하므로 동적으로 확장된다.

```bash
# Metaspace 최대 크기 설정 (설정 안 하면 시스템 메모리까지 증가)
java -XX:MaxMetaspaceSize=256m -jar app.jar
```

동적 프록시를 많이 생성하는 프레임워크(Spring AOP 등)를 쓸 때 Metaspace가 계속 증가하는 경우가 있다. 이때 `OutOfMemoryError: Metaspace`가 나온다.

### JVM Stack (Java Virtual Machine Stack)

**스레드마다 하나씩** 생성되는 영역이다. 메서드가 호출될 때마다 **Stack Frame**이 하나 push되고, 메서드가 리턴되면 pop된다.

```java
public void methodA() {
    int x = 10;        // x는 methodA의 Stack Frame에 저장
    methodB();          // methodB의 Stack Frame이 push
}

public void methodB() {
    int y = 20;        // y는 methodB의 Stack Frame에 저장
    methodC();
}

// 호출 순서: methodA → methodB → methodC
// Stack: [methodA] → [methodA][methodB] → [methodA][methodB][methodC]
// 리턴하면 역순으로 pop
```

각 Stack Frame은 다음을 포함한다:
- **Local Variables Array** — 지역 변수와 파라미터
- **Operand Stack** — 바이트코드 연산에 사용되는 스택
- **Frame Data** — Constant Pool 참조, 예외 테이블 등

Stack의 깊이가 허용 범위를 초과하면 `StackOverflowError`가 발생한다. 재귀 호출이 끝나지 않을 때 흔히 볼 수 있다.

```bash
# 스레드당 Stack 크기 설정 (기본값은 보통 512KB ~ 1MB)
java -Xss1m -jar app.jar
```

### PC Register (Program Counter Register)

각 스레드가 **현재 실행 중인 JVM 명령어의 주소**를 저장한다. 바이트코드 인터프리터가 다음에 실행할 명령어를 알기 위해 이 레지스터를 참조한다.

스레드가 Java 메서드를 실행 중이면 현재 바이트코드 명령어의 주소가 저장되고, Native 메서드를 실행 중이면 undefined 상태가 된다.

크기가 매우 작아서 OOM이 발생할 일은 없다. JVM 스펙상 이 영역에 대한 `OutOfMemoryError`는 정의되어 있지 않다.

### Native Method Stack

JNI(Java Native Interface)를 통해 호출되는 **네이티브 메서드(C/C++)를 위한 스택**이다. Java 코드가 아닌 네이티브 코드 실행을 위한 영역이다.

```java
// System.currentTimeMillis()는 내부적으로 네이티브 메서드를 호출
public static native long currentTimeMillis();

// Thread.sleep()도 네이티브 메서드
public static native void sleep(long millis) throws InterruptedException;
```

HotSpot JVM에서는 JVM Stack과 Native Method Stack을 구분하지 않고 하나로 합쳐서 관리한다. 하지만 JVM 스펙상으로는 별개의 영역이다.

## 정리: 메모리 영역과 에러 매핑

실무에서 만나는 에러와 메모리 영역의 관계를 정리하면:

| 에러 | 관련 영역 | 흔한 원인 |
|------|----------|----------|
| `OutOfMemoryError: Java heap space` | Heap | 메모리 누수, Heap 크기 부족 |
| `OutOfMemoryError: Metaspace` | Method Area | 클래스 로딩 과다, 동적 프록시 |
| `StackOverflowError` | JVM Stack | 무한 재귀, 깊은 호출 체인 |
| `OutOfMemoryError: unable to create native thread` | — | 스레드 수 초과 (OS 레벨) |

JVM 튜닝이라는 게 결국 이 메모리 영역들의 **크기와 GC 전략을 조절**하는 작업이다. 어떤 영역에서 문제가 발생하는지 알아야 올바른 옵션을 적용할 수 있다.

다음 편에서는 `.class` 파일이 이 메모리 영역들에 어떻게 로딩되고, Execution Engine이 바이트코드를 어떻게 실행하는지 정리한다.

## 참고

- [JVM Internals — James D Bloom](https://blog.jamesdbloom.com/JVMInternals.html)
- [The Java Virtual Machine Specification (Java SE 17)](https://docs.oracle.com/javase/specs/jvms/se17/html/)
