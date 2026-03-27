---
title: "모임 조회 2단계 — 쿼리 분리와 복합 인덱스로 389초 → 5초"
project: "gathering-server"
lang: ko
categories:
  - Projects
tags:
  - Java
  - Spring Boot
  - MySQL
  - EXPLAIN ANALYZE
  - Locust
date: 2026-03-19
excerpt: "enrollment 서브쿼리 분리, category JOIN 제거, 복합 인덱스 추가로 389초에서 5초로 개선했지만, ROW_NUMBER()가 풀 스캔을 강제하는 한계가 남는다."
---

> **모임 조회 API 시리즈**
> [1단계. 튜닝 없이 v1 쿼리 분석 — 389초](/projects/gathering-query-v1-no-tuning/)
> 2단계. 쿼리 분리 + 복합 인덱스 — 389초 → 5초 *(현재 글)*
> [3단계. ROW_NUMBER 제거 — 5초 → 317ms](/projects/gathering-query-v3-remove-rownum/)
> [4단계. Redis 캐시 + Redisson 분산 락 — p50 11ms](/projects/gathering-query-v4-redis-cache/)

---

## Problem

1단계에서 확인한 v1 쿼리의 핵심 병목:

1. enrollment 전체 GROUP BY가 메인 쿼리에 포함
2. category/user/image JOIN이 99만 행 전부에 발생
3. ROW_NUMBER()가 풀 스캔을 강제

이 중 1번과 2번은 **쿼리 분리**로 해결할 수 있다. 3번은 `(category_id, count)` 복합 인덱스를 추가하지만, ROW_NUMBER()가 남아있는 한 완전한 해결은 불가능하다.

## Action

### 1. enrollment 서브쿼리 분리

v1은 enrollment 전체를 GROUP BY하는 서브쿼리가 메인 쿼리에 포함되어 있었다. 이를 **별도 배치 쿼리로 분리**하여, ROW_NUMBER()로 선별된 ID에만 집계한다.

```java
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

### 2. category JOIN 제거 → category_id로 변경

v1은 `PARTITION BY ca.name`으로 카테고리명 기준 파티션을 나눴다. 이를 `category_id`(FK)로 변경하면 category JOIN 없이 gathering 테이블 단독으로 파티션을 나눌 수 있다.

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

내부 서브쿼리가 `SELECT g.*`만 처리하므로, user/image JOIN이 ROW_NUMBER() 필터링 **이후에** 발생한다. 180건에만 JOIN이 걸린다.

### 3. 카테고리명은 Java Map으로 해결

```java
Map<Long, String> categoryNameMap = categoryRepository.findAll().stream()
    .collect(Collectors.toMap(Category::getId, Category::getName));
```

카테고리는 수십 개 이하의 소규모 테이블이므로, `findAll()`로 한 번 로드하여 메모리에서 매핑하는 것이 SQL JOIN보다 효율적이다.

### 4. 복합 인덱스 추가

```sql
CREATE INDEX idx_gathering_category_count ON gathering (category_id, count DESC);
```

`(category_id, count)` 복합 인덱스를 추가했지만, ROW_NUMBER()가 전체 행에 번호를 매기는 특성상 **옵티마이저가 이 인덱스를 선택하지 않는다.**

### JdbcGatheringRepository

```java
public List<MainGatheringsProjectionV2> gatheringsV2() {
    String sql = "select g.id, g.title, g.content, g.register_date as registerDate, " +
            "g.category_id as categoryId, cr.username as createdBy, im.url as url " +
            "from ( " +
            "  select g.*, row_number() over (partition by g.category_id order by g.count desc) as rownum " +
            "  from gathering g " +
            ") g " +
            "left join user cr on g.user_id = cr.id " +
            "left join image im on g.image_id = im.id " +
            "where g.rownum between 1 and 9";
    return jdbcTemplate.query(con -> con.prepareStatement(sql), mainGatheringsV2RowMapper());
}
```

### GatheringService

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

### EXPLAIN ANALYZE

![v2 실행 전략](/assets/images/gathering/두번쨰%20실행전략.png)

- `Table scan on g` — rows=990,742 (여전히 풀 스캔)
- `Materialize` + `Window aggregate: row_number()` — actual time=4568~4568ms
- Nested loop left join — rows=180에만 user/image JOIN 발생
- 전체: **약 5초**

쿼리 분리만으로 **389초 → 5초, 약 78배** 개선됐다. 하지만 ROW_NUMBER()가 gathering 99만 행 풀 스캔을 강제하는 구조는 변하지 않았다.

### Locust 100명 부하테스트

![v2 부하테스트](/assets/images/gathering/두번쨰%20부하테스트.png)

- **0.5 RPS, 61% 요청 실패**
- p50 84,000ms / p95 84,000ms

### Grafana — HikariCP

![v2 Grafana HikariCP](/assets/images/gathering/두번째%20그라파나.png)

- Active connections: 10 (풀 전체 고갈)
- **Pending threads: 91** — 커넥션을 얻지 못해 대기하는 스레드
- 쿼리 하나가 5초 동안 커넥션을 점유하면서, 동시 100명의 요청이 줄줄이 밀리는 구조

### RDS — DBLoadCPU

![v2 DBLoadCPU](/assets/images/gathering/두번쨰%20db.png)

- DBLoadCPU: **2.47** — 풀 스캔으로 인한 CPU 부하 급등

## Reflection

하나의 SQL에 모든 것을 담으면 "한 번에 끝난다"는 편안함이 있다. 하지만 그 안에서 enrollment 전체 집계, category JOIN, user/image JOIN이 **99만 행에 중첩**되면, 각각은 작은 비용이라도 곱셈으로 폭발한다.

쿼리 분리의 핵심은 **"이 JOIN이 정말 99만 행 전부에 필요한가?"**를 묻는 것이다. enrollment count는 최종 선별된 ID에만 필요하고, 카테고리명은 수십 개짜리 테이블이다. 분리하면 각각의 비용이 독립적으로 최소화된다.

그러나 **ROW_NUMBER()가 풀 스캔을 강제하는 한, 인덱스를 추가해도 효과가 없다.** 복합 인덱스 `(category_id, count)`는 존재하지만 옵티마이저가 선택하지 않는다. 이 구조적 한계를 넘으려면 ROW_NUMBER() 자체를 제거해야 한다.

---

> ROW_NUMBER()를 제거하면 비로소 복합 인덱스가 동작한다.
> [3단계: ROW_NUMBER 제거 — 5초 → 317ms →](/projects/gathering-query-v3-remove-rownum/)
