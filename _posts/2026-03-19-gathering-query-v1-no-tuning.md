---
title: "모임 조회 1단계 — 튜닝 없이 v1 쿼리, 389초의 현실"
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
excerpt: "ROW_NUMBER() + 4개 테이블 JOIN + enrollment 서브쿼리가 중첩된 v1 쿼리가 99만 행에서 어떤 결과를 만들어내는지 분석한다."
---

> **모임 조회 API 시리즈**
> 1단계. 튜닝 없이 v1 쿼리 분석 — 389초 *(현재 글)*
> [2단계. 쿼리 분리 + 복합 인덱스 — 389초 → 5초](/projects/gathering-query-v2-separation/)
> [3단계. ROW_NUMBER 제거 — 5초 → 317ms](/projects/gathering-query-v3-remove-rownum/)
> [4단계. Redis 캐시 + Redisson 분산 락 — p50 11ms](/projects/gathering-query-v4-redis-cache/)

---

## 요구사항

모임 메인 페이지는 **카테고리별 인기 모임 9개**를 한 번에 보여줘야 한다. 카테고리가 20개라면 총 180개의 모임을 한 번의 API 호출로 반환해야 한다.

## v1 쿼리: 단일 SQL로 모든 것을 해결

v1은 이 요구사항을 **하나의 SQL**로 해결했다.

```sql
SELECT id, title, content, registerDate, category, createdBy, url, count
FROM (
  SELECT g.id, g.title, g.content, g.register_date AS registerDate,
         ca.name AS category, cr.username AS createdBy, im.url AS url,
         ec.count AS count,
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

### JdbcGatheringRepository

```java
public List<MainGatheringsProjection> gatherings() {
    String sql = "select id, title, content, registerDate, category, createdBy, url, count from ( " +
            "  select g.id as id, g.title as title, g.content as content, " +
            "         g.register_date as registerDate, ca.name as category, " +
            "         cr.username as createdBy, im.url as url, ec.count as count, " +
            "         row_number() over (partition by ca.name order by g.count desc) as rownum " +
            "  from gathering g " +
            "  left join category ca on g.category_id = ca.id " +
            "  left join user cr on g.user_id = cr.id " +
            "  left join image im on g.image_id = im.id " +
            "  left join (select count(*) as count, gathering_id " +
            "             from enrollment where accepted = true " +
            "             group by gathering_id) ec " +
            "  on ec.gathering_id = g.id" +
            ") as subquery " +
            "where rownum between 1 and 9";
    return jdbcTemplate.query(con -> con.prepareStatement(sql), mainGatheringsRowMapper());
}
```

### GatheringService

```java
public ApiResponse gatherings() {
    List<MainGatheringsProjection> mainGatheringElements = jdbcGatheringRepository.gatherings();
    Map<String, CategoryTotalGatherings> map = categorizeByCategory(mainGatheringElements);
    return toMainGatheringResponse(map);
}
```

한 번의 쿼리, 한 번의 서비스 호출. 코드만 보면 깔끔해 보인다.

## 문제 분석

하지만 이 쿼리에는 세 가지 구조적 문제가 숨어 있다.

### 1. gathering 99만 행 풀 스캔

`ROW_NUMBER()`는 **전체 행에 번호를 매긴 뒤** `WHERE rownum BETWEEN 1 AND 9`로 필터링한다. 카테고리별로 9개만 필요하지만, 99만 행을 전부 읽고 정렬해야 번호를 매길 수 있다.

### 2. enrollment 전체 GROUP BY

```sql
SELECT COUNT(*) AS count, gathering_id
FROM enrollment WHERE accepted = true
GROUP BY gathering_id
```

이 서브쿼리는 `accepted = true`인 **모든 enrollment 행**을 집계한다. 최종적으로 필요한 건 선별된 180개 모임의 enrollment count뿐인데, 전체를 GROUP BY하고 있다.

### 3. 4개 테이블이 99만 행에 JOIN

category, user, image, enrollment — 이 4개 테이블 JOIN이 ROW_NUMBER() 필터링 **이전에** 발생한다. 99만 행 전부에 조인이 걸린다.

## EXPLAIN ANALYZE 결과

![v1 실행 전략](/assets/images/gathering/첫번째%20실행전략.png)

- `Table scan on g` — rows=990,742, actual time=388,888~389,337ms
- `Nested loop left join` — 4개 테이블 중첩
- `Group aggregate: count(0)` — enrollment 전체 집계
- 전체 실행 시간: **약 389초 (6.5분)**

## Postman 결과

![v1 Postman 결과](/assets/images/gathering/첫번쨰%20postman결과.png)

단일 요청 응답 시간: **7분 38초**. API 하나가 DB 커넥션을 6분 이상 점유한다.

## Reflection

"한 번의 쿼리로 끝낸다"는 것은 코드의 단순함이지, 성능의 단순함이 아니다. ROW_NUMBER()는 "카테고리별 Top N"을 우아하게 표현하지만, 그 대가로 **99만 행 풀 스캔 + 정렬 + 4개 테이블 JOIN**이라는 비용을 숨기고 있다.

EXPLAIN ANALYZE를 실행하기 전까지는 이 쿼리가 왜 느린지 체감하기 어렵다. 실행 계획을 확인하는 습관이 모든 최적화의 출발점이다.

---

> 하나의 거대한 SQL을 역할별로 분리하면 어떻게 되는가?
> [2단계: 쿼리 분리 + 복합 인덱스 — 389초 → 5초 →](/projects/gathering-query-v2-separation/)
