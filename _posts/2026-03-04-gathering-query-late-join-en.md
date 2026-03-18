---
title: "Optimizing List Queries with the Late Join Pattern"
project: "gathering-server"
lang: en
categories:
  - Projects
tags:
  - Java
  - MySQL
  - JDBC
  - Query Tuning
date: 2026-03-04
excerpt: "Solving the full table scan problem of ROW_NUMBER() window functions with per-category separation and Late Join pattern."
---

## Problem

The `gatherings` list query used **ROW_NUMBER() window function across all categories**.

```sql
-- Before: Window function across all categories
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

With 1 million rows, **full table scan + window function cost grew linearly**. With 20 categories needing 9 items each (180 total), scanning and sorting all 1 million rows was grossly inefficient.

## Action

### Late Join Pattern

Filter with **WHERE + LIMIT first in the inner subquery**, then perform user/image joins on only 9 records.

```sql
-- After: Late Join pattern
SELECT
  g.id, g.title, g.content, g.register_date,
  g.category_name AS category,
  cr.username AS createdBy,
  im.url as url, g.count as count
FROM (
  -- Step 1: Filter with minimal data (Late Join core)
  SELECT g.*, ca.name AS category_name
  FROM gathering g
  JOIN category ca ON ca.id = g.category_id
  WHERE ca.name = ?
  ORDER BY g.count DESC
  LIMIT 9
) g
-- Step 2: Join only on 9 records
LEFT JOIN user cr ON g.user_id = cr.id
LEFT JOIN image im ON g.image_id = im.id
```

**Late Join essence**: **Narrow down targets using only PK and minimum filter conditions** in the subquery, then perform expensive joins (user, image) only on the reduced result set.

| Stage | Before | After |
|-------|--------|-------|
| Scan target | All 1M rows | Per-category filter |
| Sort target | 1M rows + window function | count DESC within category |
| Join target | user/image on 1M rows | **user/image on only 9 rows** |

## Result

- ROW_NUMBER() window function removed → **full table scan prevented**
- Late Join **minimized join targets to 9 records**
- Execution cost per category query dramatically reduced

## Reflection

The essence of Late Join is **"joins are expensive, so delay them as long as possible."** Confirm targets with PKs only in the subquery, then fetch remaining info for just the confirmed few records.

However, this improvement still had a limitation: with 20 categories, **20 queries are still needed**. Recognizing this limitation and introducing Redis cache was the next step.
