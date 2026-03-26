---
title: "모임 조회 1단계 — 쿼리 분리로 389초 → 5초"
project: "gathering-server"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - MySQL
  - EXPLAIN ANALYZE
date: 2026-03-19
excerpt: "99만 행 풀 스캔 + enrollment 서브쿼리 + 4개 테이블 내부 JOIN이 중첩된 v1 쿼리를, 쿼리 분리 전략으로 389초에서 5초로 개선한 과정."
---

> **모임 조회 API 시리즈**
> 1단계. 쿼리 분리 — 389초 → 5초 *(현재 글)*
> [2단계. 윈도우 함수 제거 + 복합 인덱스 — 5초 → 317ms](/projects/gathering-query-index/)
> [3단계. Redis 캐시 + Redisson 분산 락](/projects/gathering-query-redis-cache/)

---

## Problem

모임 메인 페이지는 카테고리별 인기 모임 9개를 한 번에 보여줘야 한다. v1 쿼리는 이 요구사항을 **단일 SQL**로 해결했다.

```sql
SELECT id, title, content, registerDate, category, createdBy, url, count
FROM (
  SELECT g.id, g.title, g.content, g.register_date AS registerDate,
         ca.name AS category, cr.username AS createdBy, im.url AS url, ec.count AS count,
         ROW_NUMBER() OVER (PARTITION BY ca.name ORDER BY g.count DESC) AS rownum
  FROM gathering g
  LEFT JOIN category ca ON g.category_id = ca.id
  LEFT JOIN user cr ON g.user_id = cr.id
  LEFT JOIN image im ON g.image_id = im.id
  LEFT JOIN (
    SELECT COUNT(*) AS count, gathering_id
    FROM enrollment WHERE accepted = true
    GROUP BY gathering_id
  ) ec ON ec.gathering_id = g.id
) AS subquery
WHERE rownum BETWEEN 1 AND 9
```

한 눈에 봐도 문제가 보인다:

1. **gathering 99만 행 풀 스캔** — `ROW_NUMBER()`가 전체 행에 번호를 매기므로, 인덱스를 걸어도 풀 스캔을 피할 수 없다.
2. **enrollment 전체 GROUP BY** — `accepted = true`인 모든 행을 집계하는 서브쿼리가 매 요청마다 실행된다.
3. **4개 테이블 내부 JOIN** — category, user, image를 ROW_NUMBER() 필터링 **이전에** JOIN하므로, 99만 행 전부에 조인이 발생한다.

EXPLAIN ANALYZE 결과는 처참했다:

![v1 실행 전략](/assets/images/gathering/첫번째%20실행전략.png)

- `Table scan on g` — rows=990,742, actual time=388,888~389,337ms
- 전체 실행 시간: **약 389초 (6.5분)**

Postman으로 확인하면 **응답 7분 38초**:

![v1 Postman 결과](/assets/images/gathering/첫번쨰%20postman결과.png)

Locust 100명 부하테스트는 더 심각했다:

![v1 부하테스트](/assets/images/gathering/두번쨰%20부하테스트.png)

- **61% 요청 실패**, p50 84초
- HikariCP 풀 10개 전부 고갈, **91개 대기 스레드** 발생

![v1 Grafana HikariCP](/assets/images/gathering/두번째%20그라파나.png)

쿼리 하나가 DB 커넥션을 6분 이상 점유하면서, 다른 요청들이 커넥션을 얻지 못해 줄줄이 실패하는 구조였다.

## Action

### 핵심 전략: 쿼리 분리

하나의 거대한 SQL에 모든 것을 맡기는 대신, **역할별로 쿼리를 분리**했다.

### 1. PARTITION BY ca.name → category_id로 변경

v1은 `PARTITION BY ca.name`으로 카테고리명 기준 파티션을 나눴다. 이를 위해 category 테이블 JOIN이 내부 서브쿼리에 필수였다.

**category_id(FK)로 변경**하면 category JOIN 없이 gathering 테이블 단독으로 파티션을 나눌 수 있다. 이는 이후 단계에서 `(category_id, count)` 복합 인덱스를 활용하기 위한 기반이기도 하다.

```sql
-- v2: category JOIN 제거, gathering 단독 서브쿼리
SELECT g.id, g.title, g.content, g.register_date AS registerDate,
       g.category_id AS categoryId, cr.username AS createdBy, im.url AS url
FROM (
  SELECT g.*, ROW_NUMBER() OVER (PARTITION BY g.category_id ORDER BY g.count DESC) AS rownum
  FROM gathering g
) g
LEFT JOIN user cr ON g.user_id = cr.id
LEFT JOIN image im ON g.image_id = im.id
WHERE g.rownum BETWEEN 1 AND 9
```

내부 서브쿼리가 `SELECT g.*`만 처리하므로, category/user/image JOIN이 모두 사라진다.

### 2. enrollment 서브쿼리 분리

v1은 enrollment 전체를 GROUP BY하는 서브쿼리가 메인 쿼리에 포함되어 있었다. 이를 **별도 배치 쿼리로 분리**하여, ROW_NUMBER()로 선별된 ID에만 집계한다.

```java
// 선별된 gathering ID에만 enrollment count 조회
Map<Long, Integer> enrollmentCounts =
    jdbcGatheringRepository.gatheringEnrollmentCounts(gatheringIds);
```

```sql
SELECT gathering_id, COUNT(*) AS count
FROM enrollment
WHERE accepted = true AND gathering_id IN (?, ?, ...)
GROUP BY gathering_id
```

99만 건 전체가 아니라, 최대 **카테고리 수 × 9건**에만 집계가 발생한다.

### 3. 카테고리명 Java Map으로 해결

```java
Map<Long, String> categoryNameMap = categoryRepository.findAll().stream()
    .collect(Collectors.toMap(Category::getId, Category::getName));
```

카테고리는 수십 개 이하의 소규모 테이블이므로, 한 번 `findAll()`로 로드하여 메모리에서 매핑하는 것이 SQL JOIN보다 효율적이다.

### 서비스 레이어: 조립

```java
public ApiResponse gatheringsV2() {
    List<MainGatheringsProjectionV2> projections = jdbcGatheringRepository.gatheringsV2();

    List<Long> gatheringIds = projections.stream()
            .map(MainGatheringsProjectionV2::getId).toList();
    Map<Long, Integer> enrollmentCounts =
            jdbcGatheringRepository.gatheringEnrollmentCounts(gatheringIds);

    Map<Long, String> categoryNameMap = categoryRepository.findAll().stream()
            .collect(Collectors.toMap(Category::getId, Category::getName));

    List<MainGatheringsProjection> mainGatheringElements = projections.stream()
            .map(p -> MainGatheringsProjection.builder()
                    .id(p.getId())
                    .title(p.getTitle())
                    .content(p.getContent())
                    .registerDate(p.getRegisterDate())
                    .category(categoryNameMap.getOrDefault(p.getCategoryId(), "unknown"))
                    .createdBy(p.getCreatedBy())
                    .url(p.getUrl())
                    .count(enrollmentCounts.getOrDefault(p.getId(), 0))
                    .build())
            .toList();

    Map<String, CategoryTotalGatherings> map = categorizeByCategory(mainGatheringElements);
    return toMainGatheringResponse(map);
}
```

## Result

![v2 실행 전략](/assets/images/gathering/두번쨰%20실행전략.png)

- EXPLAIN ANALYZE: **389초 → 5초**
- Nested loop left join — rows=180에만 user/image JOIN 발생
- `Materialize` + Window aggregate 비용: 872~3,764ms

쿼리 분리만으로 **약 78배** 개선됐다.

단, ROW_NUMBER() 특성상 gathering 99만 행 풀 스캔은 여전히 존재한다. **(category_id, count) 인덱스를 걸어도 ROW_NUMBER()가 전체 행에 번호를 매기므로 인덱스로 추가 개선이 불가능하다.**

부하테스트에서도 한계가 명확했다:
- Locust 100명: 61% 실패, p50 84초 → 여전히 사용 불가 수준
- HikariCP 풀 고갈, 91개 대기 스레드

## Reflection

하나의 SQL에 모든 것을 담으면 "한 번에 끝난다"는 편안함이 있다. 하지만 그 안에서 enrollment 전체 집계, category JOIN, user/image JOIN이 **99만 행에 중첩**되면, 각각은 작은 비용이라도 곱셈으로 폭발한다.

쿼리 분리의 핵심은 **"이 JOIN이 정말 99만 행 전부에 필요한가?"**를 묻는 것이다. enrollment count는 최종 선별된 ID에만 필요하고, 카테고리명은 수십 개짜리 테이블이다. 분리하면 각각의 비용이 독립적으로 최소화된다.

다만 389초에서 5초로 줄었어도, **ROW_NUMBER()가 풀 스캔을 강제하는 한 인덱스로는 더 이상 개선할 수 없다.** 이 구조적 한계를 넘으려면 ROW_NUMBER() 자체를 제거해야 한다.

---

> ROW_NUMBER()를 제거하면 비로소 복합 인덱스를 활용할 수 있다.
> [2단계: 윈도우 함수 제거 + 복합 인덱스 — 5초 → 317ms →](/projects/gathering-query-index/)
