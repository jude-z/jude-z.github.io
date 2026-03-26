---
title: "모임 조회 2단계 — 윈도우 함수 제거와 복합 인덱스로 5초 → 317ms"
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
excerpt: "ROW_NUMBER()를 제거하여 (category_id, count) 복합 인덱스를 활용 가능하게 전환하고, 5초에서 317ms로 개선한 과정."
---

> **모임 조회 API 시리즈**
> [1단계. 쿼리 분리 — 389초 → 5초](/projects/gathering-query-separation/)
> 2단계. 윈도우 함수 제거 + 복합 인덱스 — 5초 → 317ms *(현재 글)*
> [3단계. Redis 캐시 + Redisson 분산 락](/projects/gathering-query-redis-cache/)

---

## Problem

1단계에서 쿼리 분리로 389초 → 5초까지 개선했지만, **ROW_NUMBER()가 풀 스캔을 강제하는 구조적 한계**가 남아 있었다.

v2 쿼리의 내부 서브쿼리:

```sql
SELECT g.*, ROW_NUMBER() OVER (PARTITION BY g.category_id ORDER BY g.count DESC) AS rownum
FROM gathering g
```

`(category_id, count)` 복합 인덱스를 추가해도, ROW_NUMBER()는 **전체 행에 번호를 매긴 뒤** `WHERE rownum BETWEEN 1 AND 9`로 필터링하므로, MySQL 옵티마이저는 풀 스캔을 선택할 수밖에 없다.

EXPLAIN ANALYZE에서도 확인된다:

![v2 실행 전략](/assets/images/gathering/두번쨰%20실행전략.png)

- `Table scan on g` — rows=990,742 (여전히 풀 스캔)
- `Materialize` + Window aggregate — actual time 872~3,764ms
- 전체: **약 5초**

Locust 100명 부하테스트:
- 61% 요청 실패, p50 84초
- HikariCP 풀 고갈, 91개 대기 스레드

**인덱스를 추가하는 것만으로는 해결되지 않는다.** ROW_NUMBER() 자체를 제거해야 인덱스가 동작한다.

## Action

### 핵심 전략: ROW_NUMBER() 제거 → 카테고리별 단순 쿼리 전환

"카테고리별 인기 9개"를 구하는 데 ROW_NUMBER()가 반드시 필요한가?

카테고리별로 **개별 쿼리**를 날리면, 각 쿼리는 `WHERE + ORDER BY + LIMIT`만으로 충분하다. 이 구조는 복합 인덱스를 **100% 활용**할 수 있다.

### v3 쿼리: 카테고리별 LIMIT 9

```sql
SELECT g.id, g.title, g.content, g.register_date AS registerDate,
       g.category_id AS categoryId, cr.username AS createdBy, im.url AS url
FROM gathering g
LEFT JOIN user cr ON g.user_id = cr.id
LEFT JOIN image im ON g.image_id = im.id
WHERE g.category_id = ?
ORDER BY g.count DESC
LIMIT 9
```

`WHERE category_id = ? ORDER BY count DESC LIMIT 9` — 이 조합은 `(category_id, count)` 복합 인덱스를 타면 **인덱스 레인지 스캔으로 9건만 읽고 즉시 반환**한다.

### 복합 인덱스 추가

```sql
CREATE INDEX idx_gathering_category_count ON gathering (category_id, count DESC);
```

### 서비스 레이어: 카테고리별 반복 호출

```java
public ApiResponse gatheringsV3() {
    Map<Long, String> categoryNameMap = categoryRepository.findAll().stream()
            .collect(Collectors.toMap(Category::getId, Category::getName));

    List<MainGatheringsProjectionV2> projections = categoryNameMap.keySet().stream()
            .flatMap(categoryId ->
                jdbcGatheringRepository.gatheringsV3(categoryId).stream())
            .toList();

    List<Long> gatheringIds = projections.stream()
            .map(MainGatheringsProjectionV2::getId).toList();
    Map<Long, Integer> enrollmentCounts =
            jdbcGatheringRepository.gatheringEnrollmentCounts(gatheringIds);

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

카테고리가 20개면 쿼리 20번 + enrollment 배치 1번 = **총 21번**. ROW_NUMBER()의 99만 행 풀 스캔 대비 각 쿼리가 인덱스로 9건만 읽으므로, N번의 라운드트립 비용을 감안해도 압도적으로 빠르다.

### JdbcGatheringRepository

```java
public List<MainGatheringsProjectionV2> gatheringsV3(Long categoryId) {
    String sql = "select g.id, g.title, g.content, g.register_date as registerDate, " +
            "g.category_id as categoryId, cr.username as createdBy, im.url as url " +
            "from gathering g " +
            "left join user cr on g.user_id = cr.id " +
            "left join image im on g.image_id = im.id " +
            "where g.category_id = ? " +
            "order by g.count desc " +
            "limit 9";
    return jdbcTemplate.query(con -> {
        PreparedStatement pstmt = con.prepareStatement(sql);
        pstmt.setLong(1, categoryId);
        return pstmt;
    }, mainGatheringsV2RowMapper());
}
```

## Result

![v3 실행 전략](/assets/images/gathering/세번쨰%20실행전략.png)

- `Index lookup on g using idx_gathering_category_count` — rows=9, actual time **0.04~0.17ms**
- `Limit: 9 row(s)` — 푸시다운 적용
- 전체: **0.17ms**

**5초 → 317ms (15배 개선)**. 개별 쿼리는 0.17ms지만, 카테고리 수만큼 반복 + enrollment 배치 + 네트워크 오버헤드를 합산하면 실제 API 응답은 317ms다.

Locust 100명 부하테스트:

![v3 부하테스트](/assets/images/gathering/세번째%20부하테스트.png)

- **28 RPS, 0% 실패**
- p50 1,600ms / p95 1,900ms

이전 61% 실패에서 0% 실패로 전환됐지만, **p50 1,600ms / p95 1,900ms**로 응답 지연이 남아 있다. 카테고리 수만큼 쿼리가 발생하는 구조적 한계 — 카테고리가 늘어날수록 쿼리 수가 선형 증가한다.

## Reflection

ROW_NUMBER()는 "한 번의 쿼리로 카테고리별 Top N"을 구하는 우아한 방법이다. 하지만 **전체 행에 번호를 매기는 특성상 인덱스를 무력화**한다. 99만 행에서 9개만 필요한데, 99만 행을 전부 읽고 정렬한 뒤 9개를 고르는 것이다.

ROW_NUMBER()를 제거하고 카테고리별 단순 쿼리로 전환하면, **비로소 복합 인덱스가 동작**한다. 쿼리 횟수는 늘어나지만, 각 쿼리가 인덱스로 9건만 읽으므로 총 비용은 비교할 수 없을 만큼 작다.

다만 카테고리 수만큼 쿼리가 발생하는 구조적 한계는 쿼리 레벨에서 해결할 수 없다. 이 한계를 넘으려면 **캐시 레이어**가 필요하다.

---

> 쿼리 튜닝의 한계를 캐시로 보완한다. Redis + Redisson 분산 락 + DB fallback.
> [3단계: Redis 캐시 + Redisson 분산 락 →](/projects/gathering-query-redis-cache/)
