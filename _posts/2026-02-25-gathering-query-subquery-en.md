---
title: "gatheringDetail Query — Removing Subquery Joins with EXPLAIN ANALYZE"
project: "gathering-server"
lang: en
categories:
  - Projects
tags:
  - Java
  - MySQL
  - JDBC
  - Query Tuning
date: 2026-02-25
excerpt: "Removing unnecessary joins from subqueries and verifying execution plans with EXPLAIN ANALYZE to improve from 2s to 200ms."
---

## Problem

The `gatheringDetail` query had three structural issues:

1. **Redundant gathering rejoin in subquery** — `LEFT JOIN gathering ge ON ge.id = e.gathering_id AND ge.id = ?` — unnecessarily rejoining the gathering table inside the enrollment subquery
2. **User table join inside subquery** — `ORDER BY u.id` required the user join inside the subquery, increasing sort cost
3. **Count relied on denormalized column** — Using `g.count` directly, causing data consistency issues

EXPLAIN ANALYZE result: **Full Table Scan + Filesort, over 2 seconds.**

## Action

![Query Optimization](/assets/images/gathering/query-optimization.png)

### Before — Problematic Query

```sql
LEFT JOIN (
  SELECT e.*
  FROM enrollment e
  LEFT JOIN user u ON u.id = e.user_id
  LEFT JOIN gathering ge ON ge.id = e.gathering_id AND ge.id = ?
  ORDER BY u.id
  LIMIT 10
) e ON e.gathering_id = g.id
```

### After — Structural Improvements

**1. Remove gathering rejoin**: Simplified to `WHERE e.gathering_id = ?`

**2. Remove user join**: Changed sort key to `ORDER BY e.user_id` — `e.user_id` is an FK on the enrollment table, enabling index utilization.

**3. Separate count into dedicated aggregate subquery**: Real-time count with `accepted = true` condition.

```sql
LEFT JOIN (
  SELECT e.*
  FROM enrollment e
  WHERE e.gathering_id = ?      -- gathering rejoin removed
  ORDER BY e.user_id            -- user join unnecessary
  LIMIT 10
) e ON e.gathering_id = g.id
LEFT JOIN (
  SELECT count(*) as count, gathering_id
  FROM enrollment
  WHERE gathering_id = ? AND accepted = true
  GROUP BY gathering_id         -- separate aggregate for consistency
) ec ON ec.gathering_id = g.id
```

## Result

- EXPLAIN ANALYZE: **2s → 200ms** improvement
- Removing unnecessary subquery joins enabled **index utilization**
- Separating count into dedicated aggregation ensured **data consistency**

## Reflection

Making EXPLAIN ANALYZE a habit was the starting point for tuning. When you only see the symptom "it's slow," it's easy to think "just add an index." But **if the query structure itself prevents index utilization**, no amount of indexes will help. Join order, subquery structure, and index utilization must be **considered together** for meaningful improvement.
