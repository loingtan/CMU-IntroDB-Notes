# Lecture 12: Join Algorithms

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** October 6, 2025  
**Readings:** Database System Concepts, Chapter 15.4-15.6

---

## Join Overview

Joins combine tuples from two relations based on a predicate. They are among the most expensive operations in query processing.

### Join Types

| Join Type | Description |
|-----------|-------------|
| **Inner Join** | Only matching tuples |
| **Left Outer Join** | All tuples from left, matching from right |
| **Right Outer Join** | All tuples from right, matching from left |
| **Full Outer Join** | All tuples from both relations |
| **Cross Join** | Cartesian product |
| **Semi Join** | Left tuples with match in right |
| **Anti Join** | Left tuples with no match in right |

### Join Condition Types

| Type | Example |
|------|---------|
| **Equijoin** | `R.a = S.b` |
| **Theta Join** | `R.a < S.b` |
| **Natural Join** | Match on all common attributes |

---

## Nested Loop Join

### Simple Nested Loop Join

```
For each tuple r in R:
    For each tuple s in S:
        If r.join_key == s.join_key:
            Output (r, s)
```

**Cost:** |R| + (|R| × |S|) I/Os

**Example:**
```
R has 1000 tuples, S has 1000 tuples
Cost = 1000 + (1000 × 1000) = 1,001,000 tuple comparisons
```

**Only practical for very small tables.**

### Block Nested Loop Join

Process blocks instead of individual tuples:

```
For each block B_r in R:
    For each block B_s in S:
        For each tuple r in B_r:
            For each tuple s in B_s:
                If r.join_key == s.join_key:
                    Output (r, s)
```

**Cost:** |R| + (|R| × |S| / buffer_size) I/Os

**Example with 100-page buffer:**
```
R = 100 pages, S = 100 pages, Buffer = 10 pages
Cost = 100 + (100 × 100 / 10) = 100 + 1000 = 1100 I/Os
```

### Index Nested Loop Join

Use index on inner relation:

```
For each tuple r in R:
    Probe index on S using r.join_key
    For each matching s:
        Output (r, s)
```

**Cost:** |R| + |R| × (index lookup cost)

**Example:**
```
R = 1000 tuples
Index lookup = 3 I/Os (root → internal → leaf)
Cost = 1000 + (1000 × 3) = 4000 I/Os
```

**Excellent when one table has an index!**

---

## Sort-Merge Join

### Algorithm

```
Phase 1: Sort
------------
Sort both relations on join key (if not already sorted)

Phase 2: Merge
-------------
r = first tuple in R
s = first tuple in S

While r and s:
    If r.key < s.key:
        r = next tuple in R
    Else if r.key > s.key:
        s = next tuple in S
    Else:
        # Match found
        Output all matching combinations
        r = next tuple in R
```

### Example

```
R sorted: [1, 2, 3, 5, 7]
S sorted: [2, 3, 3, 6]

r=1, s=2: 1 < 2, advance r
r=2, s=2: Match! Output (2,2), advance s
r=2, s=3: 2 < 3, advance r
r=3, s=3: Match! Output (3,3), advance s
r=3, s=3: Match! Output (3,3), advance s
r=3, s=6: 3 < 6, advance r
r=5, s=6: 5 < 6, advance r
r=7, s=6: 7 > 6, advance s
s exhausted: Done
```

### Cost

```
Cost = Sort(R) + Sort(S) + (|R| + |S|)

If already sorted: |R| + |S|
If need to sort: 2|R|×(1 + ⌈log(|R|/M)⌉) + 2|S|×(1 + ⌈log(|S|/M)⌉) + |R| + |S|
```

### Handling Duplicates

```
When multiple tuples have same key:
  R: [5, 5, 5]
  S: [5, 5]
  
  Output: (5,5), (5,5), (5,5), (5,5), (5,5), (5,5)
  # All 3 × 2 = 6 combinations
```

---

## Hash Join

### Basic Hash Join

```
Phase 1: Build
-------------
For each row r in R (smaller table):
    hash_key = hash(r.join_column)
    Insert r into hash_table[hash_key]

Phase 2: Probe
-------------
For each row s in S (larger table):
    hash_key = hash(s.join_column)
    For each r in hash_table[hash_key]:
        If r.join_column == s.join_column:
            Output (r, s)
```

### Cost

```
Cost = 3 × (|R| + |S|) I/Os
  - Read R to build hash table
  - Read S to probe
  - (Write hash table to disk if needed)
```

### Grace Hash Join

When hash table doesn't fit in memory:

```
Phase 1: Partition
-----------------
For each row r in R:
    partition = hash(r.join_column) % N
    Write r to R_partition[partition]

For each row s in S:
    partition = hash(s.join_column) % N
    Write s to S_partition[partition]

Phase 2: Build & Probe
---------------------
For each partition i from 0 to N-1:
    Build hash table from R_partition[i]
    Probe with S_partition[i]
    Output matches
```

### Example

```
R = 100 pages, S = 1000 pages, Memory = 30 pages

Partition into 5 partitions (20 pages each):
  R → R0, R1, R2, R3, R4 (20 pages each)
  S → S0, S1, S2, S3, S4 (200 pages each)

For each partition:
  Build hash table from R_i (20 pages fits in memory)
  Probe with S_i (200 pages)
  
Total: Partition phase + 5 × (build + probe)
```

---

## Join Algorithm Selection

| Algorithm | Best For | Avoid When |
|-----------|----------|------------|
| **Nested Loop** | Very small tables | Large tables |
| **Block Nested Loop** | No indexes, small buffer | Large tables |
| **Index Nested Loop** | One table has index | No suitable index |
| **Sort-Merge** | Large tables, sorted data | Small tables |
| **Hash Join** | Large tables, equijoin | Non-equi joins |

### Decision Tree

```
Is one table very small?
  Yes → Nested Loop
  No → Is there an index on join column?
    Yes → Index Nested Loop
    No → Is data already sorted?
      Yes → Sort-Merge
      No → Is it an equijoin?
        Yes → Hash Join
        No → Sort-Merge (or Block Nested Loop)
```

---

## Summary

| Algorithm | I/O Cost | Memory | Best Case |
|-----------|----------|--------|-----------|
| **Simple Nested Loop** | \|R\| + \|R\|×\|S\| | Minimal | Never |
| **Block Nested Loop** | \|R\| + \|R\|×\|S\|/M | M pages | Small tables |
| **Index Nested Loop** | \|R\| + \|R\|×idx | Minimal | Good index |
| **Sort-Merge** | sort(R) + sort(S) + \|R\| + \|S\| | M pages | Pre-sorted |
| **Hash Join** | 3×(\|R\| + \|S\|) | \|R\| pages | Large equijoin |

---

## Key Takeaways

1. **Nested Loop**: Only for tiny tables
2. **Index Nested Loop**: Best with good indexes
3. **Sort-Merge**: Good for large tables, especially if sorted
4. **Hash Join**: Best for large equijoins
5. **Cost Model Matters**: Choose based on data characteristics

---

*Next Lecture: Query Execution I - How Queries Run*
