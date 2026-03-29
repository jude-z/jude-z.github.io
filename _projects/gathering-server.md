---
title: "Gathering Server"
description: "모임 생성·참여, 실시간 채팅, 게시판, 일정 관리를 제공하는 커뮤니티 플랫폼."
tags:
  - Java
  - Spring Boot
  - Kafka
  - Redis
  - MySQL
status: "Completed"
order: 2
slug: "gathering-server"
lang: ko
---

## Overview

모임 생성·참여, 실시간 채팅, 게시판, 알림 등을 제공하는 커뮤니티 플랫폼 백엔드. 멀티모듈 Gradle 프로젝트로 API, Domain, Infrastructure 레이어를 분리했다.

### Architecture

![Gathering Server Architecture](/assets/images/gathering/gathering-server-architecture.png)

- **api** (Spring Boot) — REST 컨트롤러, 서비스, 인증, 이미지 업로드
- **chat-server** — WebSocket(STOMP) 채팅 + Kafka 이벤트 기반 Send/Reply 분리
- **domain** — JPA 엔티티, 도메인 모델
- **infra** — Repository(JPA, QueryDSL, JDBC), Redis, Kafka 설정

### Problem Solving

이 프로젝트에서 해결한 핵심 문제들:

- **실시간 채팅 분리** — 모놀리식 채팅을 Send(실시간 응답)/Reply(영속화)로 분리, Outbox Pattern + Kafka로 메시지 유실 방지
- **Outbox Pattern** — TransactionalEventListener(BEFORE_COMMIT/AFTER_COMMIT)으로 DB-Kafka 정합성 보장, 10초 주기 재발행 스케줄러
- **모임 조회 쿼리 최적화** — EXPLAIN ANALYZE 기반 서브쿼리 개선(2초→200ms), Late Join 패턴, Redission 분산 락 기반 캐싱

### Tech Stack

| Layer | Stack |
|-------|-------|
| Backend | Java 21, Spring Boot 3.4.2, JPA, QueryDSL, JDBC |
| Messaging | Kafka (이벤트 기반 채팅), Outbox Pattern |
| WebSocket | STOMP over SockJS |
| Database | MySQL, Redis |
| Auth | JWT (HMAC-SHA), Spring Security |
| Storage | AWS S3 |
| Deploy | Docker, GitHub Actions, AWS EC2 |
