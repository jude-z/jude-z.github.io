---
title: "Trip Planner AI"
description: "여행 지역·카테고리·일수를 입력하면 AI가 최적의 일정을 추천해주는 여행 플랫폼"
tags:
  - Java
  - Spring Boot
  - Python
  - FastAPI
  - PostgreSQL
  - Redis
  - AWS S3
status: "Completed"
order: 1
slug: "trip-planner-ai"
lang: ko
---

## Overview

사용자의 여행 지역, 카테고리, 일수를 기반으로 AI가 최적의 한국 여행 일정을 추천해주는 서비스

### Architecture

![Trip Planner AI Architecture](/assets/images/trip-payment/trip-planner-architecture.png)

- **trip-core** (Spring Boot) — 인증, 여행 계획, 그룹 여행, 커뮤니티, 북마크 등 핵심 비즈니스 로직
- **trip-payment** (Spring Boot) — Toss Payments 연동, 멱등성 키 기반 결제 처리
- **fastapi-server** (FastAPI) — K-means 클러스터링 기반 추천
- **Frontend** — Vercel 배포 (React/Next.js, 별도 레포)

### Key Features

- JWT + OAuth2 인증 (Kakao, Naver, Google)
- AI 여행 일정 추천 (K-means)
- 그룹 여행 생성 및 참여
- 여행 코스 탐색 및 리뷰
- 영수증 리뷰 (S3 이미지 업로드)
- Toss 결제 연동 (멱등성 보장)

### Problem Solving

이 프로젝트에서 해결한 핵심 문제들:

- **결제 멱등성 설계** — Redis setIfAbsent + DB Idempotency 이중 구조로 단일 장애점 없는 중복 결제 차단
- **결제 유실 방지** — confirm API + PG 웹훅 + Reconciliation 스케줄러 3중 안전망으로 유실률 0% 달성
- **트랜잭션 전파 전략** — NOT_SUPPORTED + REQUIRES_NEW로 외부 PG 호출과 DB 저장을 분리, TempPayment → Payment 전환 패턴으로 결제 유실 구조적 감지

### Tech Stack

| Layer | Stack |
|-------|-------|
| Backend | Java 21, Spring Boot 3.4.3, JPA, QueryDSL |
| AI Engine | Python, FastAPI, scikit-learn, geopy |
| Database | PostgreSQL, Redis |
| Auth | JWT (HS512), Spring Security OAuth2 |
| Storage | AWS S3 |
| Payment | Toss Payments |
