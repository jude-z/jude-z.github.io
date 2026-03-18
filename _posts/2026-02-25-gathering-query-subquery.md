---
title: "gatheringDetail 쿼리 — 서브쿼리 조인 제거와 EXPLAIN ANALYZE"
project: "gathering-server"
lang: ko
categories:
  - Projects
tags:
  - Java
  - MySQL
  - JDBC
  - Query Tuning
date: 2026-02-25
excerpt: "서브쿼리 내 불필요한 조인을 제거하고, EXPLAIN ANALYZE로 실행 계획을 검증하여 2초 → 200ms로 개선한 과정."
---

## Problem

`gatheringDetail` 쿼리에 세 가지 구조적 문제가 있었다:

1. **서브쿼리 내 gathering 재조인** — `LEFT JOIN gathering ge ON ge.id = e.gathering_id AND ge.id = ?` — enrollment 서브쿼리 안에서 gathering 테이블을 불필요하게 다시 조인
2. **user 테이블 조인이 서브쿼리 안에서 발생** — `ORDER BY u.id`로 인해 user 조인이 서브쿼리 내부에서 필요, 정렬 비용 증가
3. **count가 비정규화 컬럼에 의존** — `g.count`를 직접 사용하여 정합성 문제 존재

EXPLAIN ANALYZE 결과: **Full Table Scan + Filesort, 2초 이상 소요.**

## Action

![쿼리 최적화](/assets/images/gathering/query-optimization.png)

### Before — 문제가 있던 쿼리

```sql
-- 서브쿼리 내 gathering 재조인 + user 조인
LEFT JOIN (
  SELECT e.*
  FROM enrollment e
  LEFT JOIN user u ON u.id = e.user_id
  LEFT JOIN gathering ge ON ge.id = e.gathering_id AND ge.id = ?
  ORDER BY u.id
  LIMIT 10
) e ON e.gathering_id = g.id
...
-- 비정규화 컬럼 사용
g.count as count
```

### After — 구조 개선

**1. gathering 재조인 제거**: `WHERE e.gathering_id = ?`로 단순화

서브쿼리에서 gathering 테이블을 조인할 필요가 없다. 이미 바깥에서 `WHERE g.id = ?`로 필터링하고 있으므로, enrollment만 `WHERE` 조건으로 필터링하면 된다.

**2. user 조인 제거**: `ORDER BY e.user_id`로 변경

정렬 기준을 `u.id`에서 `e.user_id`로 바꾸면 서브쿼리 내에서 user 테이블을 조인할 필요가 없다. `e.user_id`는 enrollment 테이블의 FK이므로 인덱스를 활용할 수 있다.

**3. count를 별도 집계 서브쿼리로 분리**: `accepted = true` 조건의 실시간 집계

```sql
SELECT
  g.id, g.title, g.content, g.register_date,
  ca.name as category, cr.username as createdBy,
  u.username as participatedBy, ec.count as count
FROM gathering g
JOIN category ca ON ca.id = g.category_id
JOIN user cr ON g.user_id = cr.id
JOIN image crm ON cr.image_id = crm.id
JOIN image im ON g.image_id = im.id
LEFT JOIN (
  SELECT e.*
  FROM enrollment e
  WHERE e.gathering_id = ?      -- gathering 재조인 제거
  ORDER BY e.user_id            -- user 조인 불필요
  LIMIT 10
) e ON e.gathering_id = g.id
LEFT JOIN user u ON u.id = e.user_id
LEFT JOIN image pm ON u.image_id = pm.id
LEFT JOIN (
  SELECT count(*) as count, gathering_id
  FROM enrollment
  WHERE gathering_id = ? AND accepted = true
  GROUP BY gathering_id         -- 별도 집계로 정합성 확보
) ec ON ec.gathering_id = g.id
WHERE g.id = ?
ORDER BY u.id
```

## Result

- EXPLAIN ANALYZE 기준 **2초 → 200ms** 개선
- 서브쿼리 내 불필요한 조인 제거로 **인덱스 활용** 가능
- count를 별도 집계로 분리하여 **정합성 확보**

## Reflection

EXPLAIN ANALYZE로 실행 계획을 확인하는 습관이 튜닝의 출발점이었다. "느리다"는 증상만 보면 인덱스 추가만 생각하기 쉽지만, **쿼리 구조 자체가 인덱스를 활용하지 못하는 상태**라면 인덱스를 아무리 추가해도 효과가 없다.

조인 순서, 서브쿼리 구조, 인덱스 활용을 **함께 고려해야** 실질적인 개선이 가능하다는 것을 체감했다.
