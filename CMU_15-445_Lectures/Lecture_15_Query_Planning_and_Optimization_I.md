# Lecture 15: Query Planning & Optimization I

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** October 27, 2025  
**Readings:** Database System Concepts, Chapter 16

---

## Query Optimization Overview

The query optimizer transforms a logical query plan into an efficient physical execution plan.

### Why Optimization Matters

| Query | Bad Plan | Good Plan | Difference |
|-------|----------|-----------|------------|
| Simple | 1 sec | 0.1 sec | 10x |
| Complex | 10 min | 10 sec | 60x |
| Join | Hours | Minutes | 100x+ |

### Optimization Pipeline

```
SQL Query
    ↓
Parser → Parse Tree
    ↓
Binder → Logical Plan
    ↓
Rewriter → Transformed Logical Plan
    ↓
Optimizer → Physical Plan
    ↓
Executor → Results
```

---

## Logical Plan Optimization

### Rule-Based Rewrites

Transform queries using algebraic equivalences.

#### Predicate Pushdown

Move filters closer to scans.

```sql
-- Original
SELECT * FROM (
    SELECT * FROM R WHERE a > 5
) T 
WHERE b < 10;

-- Optimized
SELECT * FROM R 
WHERE a > 5 AND b < 10;
```

**Benefit:** Filter early, reduce data movement.

#### Projection Pushdown

Eliminate unused columns early.

```sql
-- Original
SELECT a FROM (
    SELECT a, b, c, d FROM R
) T;

-- Optimized
SELECT a FROM R;
```

**Benefit:** Less data to process.

#### Join Reordering

Change order of joins for better performance.

```sql
-- Original: R ⋈ S ⋈ T
-- If R and S are large, T is small:

-- Optimized: (S ⋈ T) ⋈ R
-- Join small tables first
```

**Benefit:** Smaller intermediate results.

#### Constant Folding

Evaluate constant expressions at compile time.

```sql
-- Original
SELECT * FROM R WHERE a > 2 + 3;

-- Optimized
SELECT * FROM R WHERE a > 5;
```

#### Subquery Unnesting

Convert subqueries to joins.

```sql
-- Original (correlated subquery)
SELECT * FROM R 
WHERE a > (SELECT AVG(b) FROM S WHERE S.c = R.c);

-- Optimized (join)
SELECT R.* 
FROM R 
JOIN (SELECT c, AVG(b) as avg_b FROM S GROUP BY c) T
ON R.c = T.c 
WHERE R.a > T.avg_b;
```

---

## Cost-Based Optimization

### Cost Model

Estimate the cost of each operator.

| Factor | Units | How Measured |
|--------|-------|--------------|
| **I/O Cost** | Disk reads/writes | Page accesses |
| **CPU Cost** | Instructions | Operator complexity |
| **Memory Cost** | Buffer pages | Memory usage |
| **Network Cost** | Bytes transferred | For distributed |

### Statistics

| Statistic | Description |
|-----------|-------------|
| **Table Cardinality** | Number of rows |
| **Column Cardinality** | Number of distinct values |
| **Column Range** | Min and max values |
| **Histograms** | Value distribution |
| **Correlation Stats** | Relationships between columns |

### Selectivity Estimation

Estimate fraction of rows that match a predicate.

| Predicate | Selectivity Formula |
|-----------|---------------------|
| `= constant` | 1 / distinct_values |
| `< constant` | (constant - min) / (max - min) |
| `LIKE 'prefix%'` | 1 / distinct_prefixes |
| Join | 1 / max(distinct_values) |

#### Example

```
Column: age
Min: 18, Max: 65, Distinct: 48

Predicate: age < 30
Selectivity = (30 - 18) / (65 - 18) = 12/47 ≈ 0.255

If table has 10,000 rows:
Estimated result = 10,000 × 0.255 = 2,550 rows
```

### Histograms

More accurate than uniform distribution assumption.

```
Height-Balanced Histogram for salary:

Bucket 1: [0 - 30K]     : 1000 rows
Bucket 2: [30K - 50K]   : 1000 rows
Bucket 3: [50K - 80K]   : 1000 rows
Bucket 4: [80K - 150K]  : 1000 rows
Bucket 5: [150K - 500K] : 1000 rows

Predicate: salary > 60K
Buckets 3 (partial), 4, 5 match
Estimate: 500 + 1000 + 1000 = 2,500 rows
```

---

## Join Order Optimization

### Problem

Find optimal order to join N tables.

```
For 3 tables (A, B, C):
  (A ⋈ B) ⋈ C
  (A ⋈ C) ⋈ B
  (B ⋈ C) ⋈ A
  A ⋈ (B ⋈ C)
  B ⋈ (A ⋈ C)
  C ⋈ (A ⋈ B)
  
# of plans grows factorially with N tables
```

### Complexity

| Tables | Left-Deep Plans | Bushy Plans |
|--------|-----------------|-------------|
| 2 | 2 | 2 |
| 3 | 12 | 12 |
| 4 | 120 | 120 |
| 5 | 1,680 | 1,680 |
| 10 | ~3.6 million | ~17.6 billion |

**Problem is NP-hard!**

---

## Summary

| Topic | Key Idea |
|-------|----------|
| **Predicate Pushdown** | Filter early |
| **Projection Pushdown** | Reduce columns early |
| **Join Reordering** | Smaller intermediates |
| **Cost Model** | Estimate execution cost |
| **Statistics** | Data distribution info |
| **Selectivity** | Fraction matching predicate |

---

## Key Takeaways

1. **Rule-Based Rewrites**: Always beneficial, apply first
2. **Cost Model Drives Decisions**: Choose lowest cost plan
3. **Statistics Quality Matters**: Bad stats → bad plans
4. **Join Ordering is Critical**: Exponential search space
5. **Optimization is Heuristic**: Can't explore all plans

---

*Next Lecture: Query Planning & Optimization II - Advanced Techniques*
