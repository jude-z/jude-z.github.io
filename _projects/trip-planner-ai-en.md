---
title: "Trip Planner AI"
description: "A travel platform where AI recommends optimal itineraries based on region, categories, and duration."
tags:
  - Java
  - Spring Boot
  - Python
  - FastAPI
  - PostgreSQL
  - Redis
  - AWS S3
status: "In Progress"
order: 1
slug: "trip-planner-ai"
lang: en
---

## Overview

A full-stack platform that recommends optimal travel itineraries using AI, based on the user's travel region, categories, and number of days.

### Architecture

![Trip Planner AI Architecture](/assets/images/trip-payment/trip-planner-architecture-en.png)

- **trip-core** (Spring Boot, :8080) — Auth, trip planning, group travel, community, bookmarks, and core business logic
- **trip-payment** (Spring Boot, :8081) — Toss Payments integration with idempotency-key-based payment processing
- **fastapi-server** (FastAPI, :8000) — K-means clustering + TSP route optimization recommendation engine
- **Frontend** — Deployed on Vercel (React/Next.js, separate repo)

### Key Features

- JWT + OAuth2 authentication (Kakao, Naver, Google)
- AI travel itinerary recommendation (K-means clustering, nearest-neighbor route optimization)
- Group trip creation and participation
- Travel course browsing and reviews
- Receipt reviews (S3 image upload)
- Toss payment integration (idempotency guaranteed)

### Problem Solving

Key problems solved in this project:

- **Payment Idempotency** — Redis setIfAbsent + DB Idempotency dual-layer design eliminates SPOF for duplicate payment prevention
- **Payment Loss Prevention** — Triple safety net (confirm API + PG webhook + Reconciliation scheduler) achieves 0% payment loss rate
- **Transaction Propagation Strategy** — NOT_SUPPORTED + REQUIRES_NEW separates external PG calls from DB writes; TempPayment → Payment transition pattern enables structural detection of lost payments

### Tech Stack

| Layer | Stack |
|-------|-------|
| Backend | Java 21, Spring Boot 3.4.3, JPA, QueryDSL |
| AI Engine | Python, FastAPI, scikit-learn, geopy |
| Database | PostgreSQL, Redis |
| Auth | JWT (HS512), Spring Security OAuth2 |
| Storage | AWS S3 |
| Payment | Toss Payments |
