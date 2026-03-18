---
title: "Gathering Server"
description: "A community platform for creating gatherings, real-time chat, boards, and scheduling."
tags:
  - Java
  - Spring Boot
  - Kafka
  - Redis
  - WebSocket
  - MySQL
status: "In Progress"
order: 2
slug: "gathering-server"
lang: en
---

## Overview

A community platform backend providing gathering creation/participation, real-time chat, boards, and notifications. Multi-module Gradle project with separated API, Domain, and Infrastructure layers.

### Architecture

![Gathering Server Architecture](/assets/images/gathering/gathering-server-architecture-en.png)

- **api** (Spring Boot, :80) — REST controllers, services, auth, image upload
- **chat-server** — WebSocket (STOMP) chat + Kafka event-driven Send/Reply separation
- **domain** — JPA entities, domain models
- **infra** — Repositories (JPA, QueryDSL, JDBC), Redis, Kafka config
- **common** — Pagination, utilities, event modules

### Problem Solving

Key problems solved in this project:

- **Real-time Chat Separation** — Split monolithic chat into Send (real-time response) / Reply (persistence), achieving 0% message loss with Outbox Pattern + Kafka
- **Outbox Pattern** — TransactionalEventListener (BEFORE_COMMIT/AFTER_COMMIT) for DB-Kafka consistency, 10-second retry scheduler
- **Query Optimization** — EXPLAIN ANALYZE-based subquery improvement (2s → 200ms), Late Join pattern, Redis cache introduction

### Tech Stack

| Layer | Stack |
|-------|-------|
| Backend | Java 21, Spring Boot 3.4.2, JPA, QueryDSL, JDBC |
| Messaging | Kafka (event-driven chat), Outbox Pattern |
| WebSocket | STOMP over SockJS |
| Database | MySQL, Redis |
| Auth | JWT (HMAC-SHA), Spring Security |
| Storage | AWS S3 |
| Deploy | Docker, GitHub Actions, AWS EC2 |
