# Lecture 11: Sorting & Aggregation Algorithms

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** October 1, 2025  
**Readings:** Database System Concepts, Chapter 15.4-15.5  
**Flash Talk:** Boaz Leskes (MotherDuck)

---

## Sorting in Databases

Sorting is fundamental for:
- **ORDER BY** clauses
- **Bulk loading** B+ trees
- **Sort-merge joins**
- **Duplicate elimination** (DISTINCT)

### Why External Sort?

When data doesn't fit in memory, we need **external merge sort**.

| Data Size | Memory | Approach |
|-----------|--------|----------|
| 1 GB | 4 GB | In-memory quicksort |
| 100 GB | 4 GB | External merge sort |
| 1 TB | 4 GB | External merge sort + parallelism |

---

## External Merge Sort

### Phase 1: Sorting (Create Runs)

```
Algorithm:
1. Read M pages into memory (M = memory size / page size)
2. Sort in memory using quicksort or heapsort
3. Write sorted run to disk
4. Repeat until all data processed

Example:
  Memory: 3 pages
  Data: 12 pages
  
  Pass 1: Read pages 1-3, sort, write Run 1
  Pass 2: Read pages 4-6, sort, write Run 2
  Pass 3: Read pages 7-9, sort, write Run 3
  Pass 4: Read pages 10-12, sort, write Run 4
  
Result: 4 sorted runs on disk
```

### Phase 2: Merging (K-Way Merge)

```
Algorithm:
1. Read first page of each run into memory
2. Build min-heap of first elements
3. Extract minimum, output to result
4. Read next element from same run
5. Repeat until all runs exhausted

Example (4 runs, 3-way merge):
  Run 1: [1, 4, 7]    Run 2: [2, 5, 8]
  Run 3: [3, 6, 9]    Run 4: [10, 11, 12]
  
  Heap: [1, 2, 3] → Output 1, read 4
  Heap: [2, 3, 4] → Output 2, read 5
  Heap: [3, 4, 5] → Output 3, read 6
  ...
  Result: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
```

### Multi-Pass Merging

If too many runs for single merge:

```
Pass 1: Create sorted runs
Pass 2: Merge runs in groups → larger runs
Pass 3: Merge larger runs → final sorted output

Example: 100 runs, can merge 10 at a time
  Pass 1: 100 runs
  Pass 2: 10 runs (merged 10 at a time)
  Pass 3: 1 sorted output
```

### Cost Analysis

```
N = total pages to sort
M = pages that fit in memory

Number of runs: ⌈N/M⌉
Merge fan-in: M-1 (reserve 1 page for output)

Pass 0 (create runs): 2N I/Os (read + write)
Pass 1+ (merge): 2N I/Os per pass

Total I/Os: 2N × (1 + ⌈log_{M-1}(N/M)⌉)
```

---

## Optimizations

### Double Buffering

Prefetch next page while processing current:

```
Buffer 1: Processing
Buffer 2: Prefetching

When Buffer 1 done:
  Swap buffers
  Start processing Buffer 2
  Start prefetching next into Buffer 1
```

**Benefit:** Overlaps I/O with computation

### Replacement Selection

Produces runs **larger than memory** using a heap.

```
Algorithm:
1. Read M pages into memory
2. Build min-heap
3. Extract minimum, output to run
4. Read next record from input
   - If key ≥ last output key: insert into heap
   - Else: save for next run (dead space)
5. Repeat until heap empty
6. Start new run with saved records

Result: Average run size = 2M
```

**Benefit:** Fewer runs → fewer merge passes

---

## Aggregation Algorithms

### Sorting-Based Aggregation

```
Algorithm:
1. Sort data on GROUP BY key(s)
2. Scan sorted data, maintaining running aggregates
3. Output group when key changes

Example:
  Input: [(A, 10), (B, 20), (A, 30), (B, 40), (C, 50)]
  
  Sort: [(A, 10), (A, 30), (B, 20), (B, 40), (C, 50)]
  
  Scan:
    Group A: sum = 10 + 30 = 40
    Group B: sum = 20 + 40 = 60
    Group C: sum = 50
```

### Hash-Based Aggregation

```
Algorithm:
1. Build hash table on GROUP BY key(s)
2. For each input row:
   - Hash the key
   - If key in hash table: update aggregate
   - Else: insert new entry
3. Output all hash entries

Example:
  Input: [(A, 10), (B, 20), (A, 30), (B, 40), (C, 50)]
  
  Hash Table:
    A: sum = 40
    B: sum = 60
    C: sum = 50
```

### Comparison

| Aspect | Sort-Based | Hash-Based |
|--------|-----------|------------|
| Requires sorting | Yes | No |
| Memory usage | Moderate | Can be high |
| Large data | External sort | Partitioning |
| Output order | Sorted | Unsorted |
| DISTINCT | Natural | Needs extra step |
| Multiple aggregates | Efficient | Efficient |

---

## Parallel Aggregation

### Partitioned Aggregation

```
Phase 1: Local Aggregation
-------------------------
Each thread:
  - Processes its partition of input
  - Builds local hash table
  - Outputs partial aggregates

Phase 2: Global Aggregation
-------------------------
Combine partial aggregates:
  - Hash by group key
  - Merge aggregates for same group

Example:
  Thread 1: A=10, B=20
  Thread 2: A=30, C=50
  Thread 3: B=40, C=60
  
  Global: A=40, B=60, C=110
```

### Two-Phase Aggregation

```
Phase 1 (Local):
  SELECT dept, SUM(salary) as local_sum
  FROM employees
  GROUP BY dept
  -- Run on each node/thread

Phase 2 (Global):
  SELECT dept, SUM(local_sum) as total_sum
  FROM local_results
  GROUP BY dept
```

---

## Summary

| Algorithm | Use Case | Key Idea |
|-----------|----------|----------|
| **External Merge Sort** | Large datasets | Create runs, then merge |
| **Replacement Selection** | Better run sizes | Heap-based, runs = 2M |
| **Sort-Based Aggregation** | When sorted output needed | Sort then scan |
| **Hash-Based Aggregation** | General case | Hash table for groups |
| **Parallel Aggregation** | Multi-core/distributed | Partition then combine |

---

## Key Takeaways

1. **External Sort**: Essential for data larger than memory
2. **Multi-Way Merge**: Reduces number of passes
3. **Double Buffering**: Overlaps I/O with computation
4. **Hash Aggregation**: Usually faster than sort-based
5. **Parallelism**: Partition work, combine results

---

*Next Lecture: Join Algorithms - Combining Relations Efficiently*
