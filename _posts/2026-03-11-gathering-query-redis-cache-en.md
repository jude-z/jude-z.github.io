---
title: "Redis Cache — Compensating for Query Tuning's Structural Limits"
project: "gathering-server"
lang: en
categories:
  - Projects
tags:
  - Java
  - Redis
  - Spring Cache
date: 2026-03-11
excerpt: "When subquery optimization and Late Join weren't enough, Redis cache addressed the structural limitation of N queries per N categories."
---

## Problem

Subquery improvement (2s → 200ms) and Late Join dramatically improved individual query efficiency, but a **structural limitation** remained.

With 20 categories, **20 queries are executed**. No matter how fast each query is, the number of queries growing proportionally with category count leads to DB load as data and traffic increase.

> "Query tuning alone cannot fully solve performance degradation as data grows."

Recognizing this limitation was the starting point for cache introduction.

## Action

### Spring Cache + Redis

Cache query results in Redis to **eliminate repetitive DB queries**.

```java
@Configuration
@EnableCaching
public class CacheConfig {
}
```

### Cache Strategy Rationale

Gathering list queries have **overwhelmingly high read ratios with relatively infrequent data changes** — ideal characteristics for caching.

| Characteristic | Value | Cache Suitability |
|---------------|-------|-------------------|
| Read:Write ratio | ~100:1 | High |
| Data change frequency | On gathering create/update | Low |
| Acceptable staleness | Several minutes | High |

### Optimization Summary Across All Steps

```
Step 1: EXPLAIN ANALYZE-based subquery restructuring
  → 2s → 200ms (index utilization)

Step 2: ROW_NUMBER() → per-category Late Join
  → Join target minimized (1M rows → 9 rows)

Step 3: Redis Cache introduction
  → DB load eliminated + consistent response times
```

## Result

- Query indexing improved **2s → 200ms**
- Late Join + Redis cache **eliminated DB load**
- **Consistent response times** regardless of data growth
- Cache expiration policy maintains **data consistency**

## Reflection

Four lessons from the three stages of query tuning:

1. **EXPLAIN ANALYZE is the starting point** — Root causes of slow queries must be found in execution plans, not guesswork
2. **Fixing structure revives indexes** — Once unnecessary subquery joins were removed, indexes started being utilized
3. **Query tuning has limits** — Even with Late Join optimizing each query, the fundamental limitation of N queries for N categories can't be solved at the query level
4. **Recognizing limits and adding layers is key** — Cache doesn't "make queries faster"; it "eliminates queries entirely"

Identifying root causes and solving them structurally. Outbox Pattern, Late Join, EXPLAIN ANALYZE were the results of that process.
