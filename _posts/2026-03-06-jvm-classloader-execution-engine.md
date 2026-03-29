---
title: "JVM 클래스 로딩 과정과 실행 엔진 동작 원리"
categories:
  - Dev Notes
tags:
  - JVM
  - Java
lang: ko
date: 2026-03-17
excerpt: "Class Loader의 Loading → Linking → Initialization 과정과 Execution Engine의 Interpreter, JIT Compiler, GC를 정리한다."
---

> **JVM Internals 시리즈**
> [1편. JVM 아키텍처와 런타임 데이터 영역](/dev%20notes/jvm-architecture-runtime-data/)
> 2편. 클래스 로딩과 실행 엔진 *(현재 글)*
> [3편. JVM 스레드와 스택 프레임 구조](/dev%20notes/jvm-thread-stack-frame/)

---

## 클래스는 언제 메모리에 올라가는가

지난 편에서 Runtime Data Areas를 정리했는데, 그럼 `.class` 파일이 이 메모리 영역에 **어떤 과정을 거쳐서 올라가는지**가 이번 주제다.

Java에서 `new UserService()`를 호출하면, JVM은 먼저 `UserService` 클래스가 이미 로딩되어 있는지 확인한다. 로딩되어 있지 않으면 Class Loader가 작동한다.

![Class Loading & Execution Engine](/assets/images/jvm/jvm-classloader-execution.png)

## Class Loading Process

클래스 로딩은 세 단계를 거친다: **Loading → Linking → Initialization**

### 1단계: Loading

`.class` 파일의 바이트코드를 읽어서 Method Area에 저장하는 단계다. 이때 해당 클래스를 나타내는 `java.lang.Class` 객체가 Heap에 생성된다.

```java
// 이 코드를 실행하면 Loading 단계에서 만들어진 Class 객체를 얻는다
Class<?> clazz = UserService.class;
// 또는
Class<?> clazz = Class.forName("com.example.UserService");
```

Loading은 세 가지 종류의 Class Loader가 계층적으로 처리한다:

1. **Bootstrap Class Loader** — `java.lang.*`, `java.util.*` 같은 핵심 Java API 로딩. C/C++로 구현되어 있어서 Java 코드에서 참조하면 null이 나온다.
2. **Extension (Platform) Class Loader** — `$JAVA_HOME/lib/ext` 디렉토리의 클래스 로딩. Java 9부터는 Platform Class Loader로 이름이 바뀜.
3. **Application (System) Class Loader** — classpath에 있는 애플리케이션 클래스 로딩. 우리가 작성한 코드는 대부분 여기서 로딩된다.

```java
public class ClassLoaderTest {
    public static void main(String[] args) {
        // Application ClassLoader
        System.out.println(ClassLoaderTest.class.getClassLoader());
        // sun.misc.Launcher$AppClassLoader@...

        // Bootstrap ClassLoader (null로 표시)
        System.out.println(String.class.getClassLoader());
        // null

        // 부모 관계 확인
        ClassLoader appLoader = ClassLoaderTest.class.getClassLoader();
        System.out.println(appLoader.getParent());
        // sun.misc.Launcher$ExtClassLoader@...
        System.out.println(appLoader.getParent().getParent());
        // null (Bootstrap)
    }
}
```

### Parent Delegation Model (부모 위임 모델)

클래스 로딩 요청이 들어오면, **자기가 직접 로딩하지 않고 먼저 부모 Class Loader에게 위임**한다. 부모가 못 찾으면 그때 자기가 로딩을 시도한다.

```
요청: "com.example.UserService 로딩해줘"
Application ClassLoader → "부모한테 먼저 물어볼게"
  → Extension ClassLoader → "나도 부모한테 물어볼게"
    → Bootstrap ClassLoader → "나한테 없음"
  → Extension ClassLoader → "나한테도 없음"
→ Application ClassLoader → "내가 로딩함" ✓
```

이 모델의 목적은 **핵심 Java 클래스를 보호**하는 것이다. 만약 누군가 `java.lang.String` 클래스를 직접 만들어서 classpath에 넣어도, Bootstrap Class Loader가 먼저 진짜 String을 로딩하기 때문에 위조된 클래스는 무시된다.

### 2단계: Linking

Linking은 다시 세 하위 단계로 나뉜다:

**Verification (검증)**
- 로딩된 바이트코드가 JVM 스펙에 맞는지 검사한다
- 매직 넘버(`0xCAFEBABE`)가 맞는지, 바이트코드 구조가 유효한지 등
- 잘못된 `.class` 파일이 JVM을 크래시시키는 걸 방지한다

```bash
# .class 파일의 매직 넘버 확인
xxd MyClass.class | head -1
# 00000000: cafe babe 0000 0034 ...
```

**Preparation (준비)**
- static 변수에 **기본값**을 할당한다 (실제 초기값이 아님!)
- `int`는 0, `boolean`은 false, 참조 타입은 null

```java
public class Config {
    static int maxRetry = 5;  // Preparation 단계에서는 0으로 초기화
                               // 5가 할당되는 건 Initialization 단계
    static boolean enabled = true;  // 여기서는 false
}
```

**Resolution (해석)**
- 심볼릭 레퍼런스를 실제 메모리 주소(다이렉트 레퍼런스)로 변환한다
- Constant Pool의 심볼릭 참조가 실제 메모리 참조로 바뀌는 단계
- 이 단계는 필요할 때(lazy) 수행될 수도 있다

### 3단계: Initialization

**실제 초기값 할당**이 이루어지는 단계다. static 변수에 코드에서 지정한 값이 할당되고, static 블록이 실행된다.

```java
public class DatabaseConfig {
    // Initialization 단계에서 "jdbc:mysql://localhost:3306/mydb"가 할당
    static String url = "jdbc:mysql://localhost:3306/mydb";

    static {
        // static 초기화 블록도 이 단계에서 실행
        System.out.println("DatabaseConfig initialized");
    }
}
```

Initialization은 JVM이 보장하는 **thread-safe** 작업이다. 여러 스레드가 동시에 같은 클래스의 초기화를 시도해도, 실제로 초기화는 한 번만 수행된다. 이게 Singleton 패턴에서 `static inner class`를 활용하는 이유이기도 하다.

```java
// thread-safe한 Singleton (클래스 로딩의 Initialization 보장을 활용)
public class Singleton {
    private Singleton() {}

    private static class Holder {
        // Holder 클래스가 최초 참조될 때 한 번만 초기화
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;  // Holder 클래스 로딩 트리거
    }
}
```

## Execution Engine

클래스가 메모리에 올라갔으면 이제 **실행**할 차례다. Execution Engine은 Method Area에 있는 바이트코드를 읽어서 실행한다.

### Interpreter

바이트코드 명령어를 **한 줄씩** 해석하고 실행한다. 시작 속도가 빠르지만, 같은 메서드를 반복 호출해도 매번 해석하기 때문에 전체적인 성능은 떨어진다.

```java
// 이 메서드가 10000번 호출되면, Interpreter는 매번 바이트코드를 해석한다
public int add(int a, int b) {
    return a + b;
}
```

### JIT Compiler (Just-In-Time Compiler)

Interpreter의 성능 문제를 보완한다. **자주 실행되는 코드(Hot Code)**를 감지해서 네이티브 코드로 컴파일한다. 한 번 컴파일되면 이후에는 네이티브 코드를 직접 실행하므로 훨씬 빠르다.

```bash
# JIT 컴파일 로그 확인
java -XX:+PrintCompilation -jar app.jar

# 출력 예시:
# 100   1       java.lang.String::hashCode (55 bytes)
# 110   2       java.lang.String::equals (81 bytes)
```

HotSpot JVM에는 두 가지 JIT 컴파일러가 있다:

- **C1 (Client Compiler)** — 빠르게 컴파일하지만 최적화 수준이 낮음. 시작 속도가 중요한 클라이언트 애플리케이션에 적합.
- **C2 (Server Compiler)** — 컴파일은 느리지만 고도로 최적화된 코드를 생성. 장시간 실행되는 서버 애플리케이션에 적합.

현대 JVM은 **Tiered Compilation**을 사용한다. 처음에는 Interpreter로 시작하고, 자주 호출되는 메서드는 C1으로 먼저 컴파일, 더 자주 호출되면 C2로 재컴파일한다.

```bash
# Tiered Compilation (Java 8부터 기본 활성화)
java -XX:+TieredCompilation -jar app.jar

# 컴파일 임계값 확인 (기본 10000)
java -XX:+PrintFlagsFinal -version | grep CompileThreshold
```

JIT가 수행하는 대표적인 최적화:
- **Inlining** — 메서드 호출을 메서드 본문으로 대체
- **Dead Code Elimination** — 실행되지 않는 코드 제거
- **Loop Unrolling** — 루프 반복을 펼쳐서 브랜치 오버헤드 제거
- **Escape Analysis** — 객체가 메서드 밖으로 나가지 않으면 스택에 할당

### Garbage Collector (GC)

Heap에서 **더 이상 참조되지 않는 객체**를 찾아서 메모리를 회수한다. 이건 주제가 너무 넓어서 별도로 정리할 내용이지만, Execution Engine의 구성 요소라는 점은 알고 있어야 한다.

간단하게만 짚으면:

```bash
# GC 종류 선택
java -XX:+UseG1GC -jar app.jar        # G1 GC (Java 9+ 기본)
java -XX:+UseZGC -jar app.jar          # ZGC (저지연)
java -XX:+UseShenandoahGC -jar app.jar # Shenandoah GC
```

GC가 동작할 때 **Stop-the-World(STW)** 이벤트가 발생한다. 모든 애플리케이션 스레드가 멈추고 GC 스레드만 작동한다. 최신 GC(ZGC, Shenandoah)는 이 STW 시간을 최소화하는 데 초점을 맞추고 있다.

## 바이트코드 실행 흐름 요약

전체 흐름을 하나로 이으면 이렇다:

```
.java 파일
  → javac (컴파일) → .class 파일 (바이트코드)
  → Class Loader (Loading → Linking → Initialization)
  → Method Area에 클래스 정보 저장
  → Execution Engine (Interpreter / JIT)이 바이트코드 실행
  → GC가 Heap의 미사용 객체 정리
```

JVM의 강점은 이 구조 덕분에 **Write Once, Run Anywhere**가 가능하다는 것이다. `.class` 파일은 플랫폼에 독립적이고, 각 OS별 JVM이 해당 플랫폼에 맞게 실행을 처리한다.

다음 편에서는 JVM 내부의 스레드 종류와 Stack Frame의 세부 구조를 정리한다.

## 참고

- [JVM Internals — James D Bloom](https://blog.jamesdbloom.com/JVMInternals.html)
- [The Java Virtual Machine Specification (Java SE 17)](https://docs.oracle.com/javase/specs/jvms/se17/html/)
