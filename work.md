# JVM Dev Notes 작업 현황

## 완료된 작업

### 1. 참고 자료 수집 (완료)
- [x] JVM Internals - 아키텍처, 런타임 데이터, 클래스 로딩, 실행 엔진, 스레드/스택 프레임
  - https://blog.jamesdbloom.com/JVMInternals.html#jvm_system_threads
- [x] GC 관련 - G1 GC, ZGC, 3색 마킹, Region, Colored Pointer
  - https://www.alibabacloud.com/blog/601536
- [x] Lock 관련 - 락 에스컬레이션, synchronized, ReentrantLock, AQS
  - https://www.alibabacloud.com/blog/lets-talk-about-several-of-the-jvm-level-locks-in-java_596090

### 2. Figma 다이어그램 생성 (완료)
Figma 파일: `architecture` (fileKey: r6TUEk0QwO0iyrZ2i8ndxV)
섹션: "JVM Dev Notes Architecture" (nodeId: 64:901)

| # | 프레임 이름 | Node ID | 용도 |
|---|------------|---------|------|
| 1 | JVM-Architecture-Overview | 64:990 | JVM 아키텍처 전체 구조도 |
| 2 | JVM-ClassLoader-Execution | 64:1062 | 클래스 로딩 과정 & 실행 엔진 |
| 3 | JVM-Thread-Stack-Frame | 64:1215 | 스레드 구조 & 스택 프레임 |
| 4 | G1-GC-Architecture | 64:1295 | G1 GC Region 메모리 레이아웃 |
| 5 | ZGC-Architecture | 64:1420 | ZGC Colored Pointer & 수집 과정 |
| 6 | Lock-Escalation | 64:1502 | synchronized 락 에스컬레이션 & Object Header |
| 7 | ReentrantLock-AQS-Concurrency | 64:1579 | ReentrantLock, AQS, 동시성 제어 |

---

## 남은 작업

### 3. Figma 스크린샷 내보내기 (미완료)
- [ ] 7개 프레임 각각 `figma_take_screenshot`으로 URL 획득 (Rate limit 해소 후)
- [ ] curl로 PNG 다운로드 → `assets/images/jvm/` 디렉토리에 저장
  - `jvm-architecture-overview.png`
  - `jvm-classloader-execution.png`
  - `jvm-thread-stack-frame.png`
  - `g1-gc-architecture.png`
  - `zgc-architecture.png`
  - `lock-escalation.png`
  - `reentrantlock-aqs.png`

### 4. 블로그 글 작성 (미완료) - 총 7개
모두 한글, DevNote 카테고리 (`project` 필드 없음), AI 티 안 나게 작성

#### JVM 글 3개 (날짜: ~3주 전, 2026-03-05~07)
- [ ] `_posts/2026-03-05-jvm-architecture-runtime-data.md`
  - JVM 아키텍처 개요, 런타임 데이터 영역 (Heap, Stack, Method Area, PC Register)
  - 이미지: jvm-architecture-overview.png

- [ ] `_posts/2026-03-06-jvm-classloader-execution-engine.md`
  - 클래스 로딩 과정 (Loading → Linking → Initialization), 실행 엔진 (Interpreter, JIT)
  - 이미지: jvm-classloader-execution.png

- [ ] `_posts/2026-03-07-jvm-thread-stack-frame.md`
  - JVM 시스템 스레드, 스택 프레임 구조 (Local Variables, Operand Stack, Frame Data)
  - 이미지: jvm-thread-stack-frame.png

#### GC 글 2개 (날짜: ~2주 전, 2026-03-12~13)
- [ ] `_posts/2026-03-12-g1-gc-region-memory-layout.md`
  - G1 GC Region 기반 메모리, RSet, Card Table, Young/Mixed GC, SATB
  - 이미지: g1-gc-architecture.png

- [ ] `_posts/2026-03-13-zgc-colored-pointer-concurrent.md`
  - ZGC Page 레이아웃, Colored Pointer, Read Barrier, self-healing, 수집 단계
  - 이미지: zgc-architecture.png

#### 스레드 락 글 2개 (날짜: ~1주 전, 2026-03-19~20)
- [ ] `_posts/2026-03-19-synchronized-lock-escalation.md`
  - Object Header/Mark Word, 편향 락 → 경량 락 → 중량 락 에스컬레이션, Monitor 구조
  - 이미지: lock-escalation.png

- [ ] `_posts/2026-03-20-reentrantlock-aqs-concurrency.md`
  - ReentrantLock vs synchronized, AQS/CLH Queue, ReadWriteLock, LongAdder, 락 최적화
  - 이미지: reentrantlock-aqs.png

### 5. Front Matter 형식
```yaml
---
title: "제목"
categories:
  - Dev Notes
tags:
  - JVM  # or GC, Concurrency 등
lang: ko
date: 2026-03-XX
excerpt: "한줄 요약"
---
```

### 6. 최종 확인
- [ ] `mkdir -p assets/images/jvm/` 디렉토리 생성
- [ ] 이미지 파일 다운로드 완료 확인
- [ ] 7개 포스트 파일 생성 확인
- [ ] 각 포스트에 이미지 경로 올바른지 확인
