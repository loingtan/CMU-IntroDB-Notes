# Lecture 16: Query Planning & Optimization II

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** October 29, 2025  
**Readings:** Database System Concepts, Chapter 16  
**[📹 Watch Video](https://www.youtube.com/watch?v=azTHRpzl10o&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=16)**

---

## Dynamic Programming for Join Ordering

### System R Algorithm

Build optimal plans bottom-up by size.

```
Algorithm:
For each single table:
    Compute best access path (scan vs index)

For size from 2 to N:
    For each subset of tables with this size:
        For each way to split subset into two:
            cost = cost(left) + cost(right) + join_cost(left, right)
            Keep minimum cost plan
```

### Example

```
Tables: A (100 rows), B (200 rows), C (300 rows)

Size 1:
  {A}: scan cost = 100
  {B}: scan cost = 200
  {C}: scan cost = 300

Size 2:
  {A,B}: 
    (A ⋈ B): 100 + 200 + join(100,200) = 500
    (B ⋈ A): 200 + 100 + join(200,100) = 500
    Best: 500
  
  {A,C}: 100 + 300 + join(100,300) = 700
  {B,C}: 200 + 300 + join(200,300) = 900

Size 3:
  {A,B,C}:
    ({A,B} ⋈ C): 500 + 300 + join(?,300)
    ({A,C} ⋈ B): 700 + 200 + join(?,200)
    ({B,C} ⋈ A): 900 + 100 + join(?,100)
    Best: minimum of above
```

### Pruning

Don't keep all plans—prune dominated ones.

```
For same set of tables, keep plan P only if:
  - P has lowest cost, OR
  - P has interesting order (for future joins)

Interesting orders: sorted on join columns
```

---

## Greedy Heuristics

When DP is too expensive, use greedy approaches.

### Greedy Join Ordering

```
Algorithm:
1. Start with single table (smallest or most selective)
2. At each step, add table that minimizes cost increase
3. Continue until all tables joined

Example:
  Tables: A (100), B (200), C (300), D (400)
  
  Start: A (smallest)
  Add B: cost(A⋈B) = 300
  Add C: cost((A⋈B)⋈C) = 300 + 300 = 600
  Add D: cost(((A⋈B)⋈C)⋈D) = 600 + 400 = 1000
```

**Advantages:**
- Fast: O(N²) instead of exponential
- Works for many tables

**Disadvantages:**
- May miss optimal plan
- No guarantee of quality

---

## Genetic Algorithms

Used by PostgreSQL for many tables (>12).

### Approach

```
1. Create initial population of random plans
2. Evaluate fitness (cost) of each plan
3. Select best plans as parents
4. Combine parents (crossover) to create offspring
5. Mutate some offspring randomly
6. Repeat for N generations
7. Return best plan found
```

**Benefit:** Can explore large search space without exhaustive search.

---

## Physical Plan Selection

For each logical operator, choose physical implementation.

### Scan Selection

| Logical | Physical Options | When to Use |
|---------|-----------------|-------------|
| Table Scan | Sequential Scan | No index, large range |
| | Index Scan | Selective predicate |
| | Index-Only Scan | All columns in index |

### Join Selection

| Logical | Physical Options | When to Use |
|---------|-----------------|-------------|
| Join | Nested Loop | Small tables |
| | Hash Join | Large tables, equijoin |
| | Sort-Merge Join | Sorted data, non-equi |
| | Index Nested Loop | Good index exists |

### Example Decision

```sql
SELECT * FROM A JOIN B ON A.x = B.y WHERE A.z > 100
```

```
Options:
1. SeqScan(A) ⋈ HashJoin SeqScan(B)
2. IndexScan(A, z > 100) ⋈ NestedLoop IndexScan(B)
3. SeqScan(A) ⋈ MergeJoin Sort(SeqScan(B))

Optimizer estimates costs, picks minimum.
```

---

## Adaptive Query Optimization

### Problem

Statistics may be inaccurate.

```
Optimizer estimates: 100 rows
Actual: 1,000,000 rows

Bad plan chosen → Slow execution
```

### Runtime Statistics

```
During execution:
  - Collect actual cardinalities
  - Compare with estimates
  - Re-optimize if significantly different
```

### Plan Re-optimization

```
Execute partial plan
  ↓
Collect statistics
  ↓
If estimates wrong by threshold:
  - Pause execution
  - Re-optimize remaining plan
  - Continue with new plan
  ↓
Complete execution
```

### Parametric Query Optimization

Cache plans for different parameter ranges.

```sql
-- Query with parameter
SELECT * FROM orders WHERE customer_id = ?

Different plans for:
- ? = specific value (index scan)
- ? = range (sequential scan)
```

---

## Summary

| Technique | Use Case |
|-----------|----------|
| **Dynamic Programming** | Small number of tables (<10) |
| **Greedy Heuristics** | Medium number of tables |
| **Genetic Algorithms** | Large number of tables (>12) |
| **Adaptive Execution** | Uncertain statistics |
| **Parametric Optimization** | Repeated queries with parameters |

---

## Key Takeaways

1. **DP is Optimal but Expensive**: Use for small queries
2. **Heuristics Scale Better**: Accept suboptimal for speed
3. **Physical Selection Matters**: Choose right algorithm
4. **Adaptivity Handles Uncertainty**: Runtime adjustments
5. **No Perfect Optimizer**: Trade-offs everywhere

---

*Next Lecture: Concurrency Control Theory*
