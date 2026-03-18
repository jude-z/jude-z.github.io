---
title: "Late Join 패턴으로 목록 조회 최적화"
project: "gathering-server"
lang: ko
categories:
  - Projects
tags:
  - Java
  - MySQL
  - JDBC
  - Query Tuning
date: 2026-03-04
excerpt: "ROW_NUMBER() 윈도우 함수의 전체 테이블 스캔 문제를 카테고리별 분리 + Late Join 패턴으로 해결한 과정."
---

## Problem

`gatherings` 목록 조회 쿼리가 **전체 카테고리를 대상으로 ROW_NUMBER() 윈도우 함수**를 사용하고 있었다.

```sql
-- Before: 전체 카테고리 대상 윈도우 함수
SELECT id, title, content, ...
FROM (
  SELECT g.*, ca.name as category,
    ROW_NUMBER() OVER (PARTITION BY ca.name ORDER BY g.count DESC) as rownum
  FROM gathering g
  LEFT JOIN category ca ON g.category_id = ca.id
  LEFT JOIN user cr ON g.user_id = cr.id
  LEFT JOIN image im ON g.image_id = im.id
) AS subquery
WHERE rownum BETWEEN 1 AND 9
```

데이터가 100만 건일 때, **전체 테이블 스캔 + 윈도우 함수 비용이 선형 증가**하는 구조적 한계가 있었다. 카테고리가 20개면 카테고리당 9건씩 총 180건만 필요한데, 100만 건 전체를 스캔하고 정렬하는 것은 비효율적이다.

## Action

### Step 1 — 카테고리별 분리 호출

전체 카테고리를 한 번에 조회하는 대신, **카테고리별로 분리 호출**한다.

```java
// Before (주석 처리된 코드)
// List<GatheringsProjection> gatheringsProjections = CategoryUtil.list.stream()
//     .map(jdbcGatheringRepository::subGatherings)
//     .flatMap(List::stream)
//     .map(GatheringsProjection::of)
//     .toList();

// After
List<MainGatheringsProjection> mainGatheringElements =
    jdbcGatheringRepository.gatherings();
```

### Step 2 — Late Join 패턴 적용

내부 서브쿼리에서 **WHERE + LIMIT으로 먼저 9건을 필터링**한 후, user/image 조인은 9건으로 줄인 뒤에 수행한다.

```sql
-- After: Late Join 패턴
SELECT
  g.id, g.title, g.content, g.register_date,
  g.category_name AS category,
  cr.username AS createdBy,
  im.url as url,
  g.count as count
FROM (
  -- 1단계: 최소 데이터로 필터링 (Late Join의 핵심)
  SELECT g.*, ca.name AS category_name
  FROM gathering g
  JOIN category ca ON ca.id = g.category_id
  WHERE ca.name = ?
  ORDER BY g.count DESC
  LIMIT 9
) g
-- 2단계: 9건에 대해서만 조인
LEFT JOIN user cr ON g.user_id = cr.id
LEFT JOIN image im ON g.image_id = im.id
```

**Late Join의 핵심**: 서브쿼리에서 **PK와 최소 필터 조건만으로 대상을 먼저 줄이고**, 무거운 조인(user, image)은 줄어든 결과에 대해서만 수행한다.

| 단계 | Before | After |
|------|--------|-------|
| 스캔 대상 | 100만 건 전체 | 카테고리별 필터링 |
| 정렬 대상 | 100만 건 + 윈도우 함수 | 카테고리 내 count DESC |
| 조인 대상 | 100만 건에 user/image 조인 | **9건에만 user/image 조인** |

## Result

- ROW_NUMBER() 윈도우 함수 제거 → **전체 테이블 스캔 방지**
- Late Join으로 **조인 대상을 9건으로 최소화**
- 각 카테고리 쿼리의 실행 비용 대폭 감소

## Reflection

Late Join 패턴의 본질은 "**조인은 비싸니까 최대한 늦게 하자**"이다. 서브쿼리에서 PK만으로 대상을 먼저 확정하고, 나머지 정보는 확정된 소수의 레코드에 대해서만 가져온다.

하지만 이 개선에도 한계가 있었다. 카테고리가 20개면 **20번의 쿼리가 발생**하는 구조적 문제가 남아 있었다. 이 한계를 인식하고 Redis 캐시를 도입한 것이 다음 단계의 개선이었다.
