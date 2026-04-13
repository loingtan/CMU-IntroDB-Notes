# Lecture 10: Index Concurrency Control

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** September 29, 2025  
**Readings:** Database System Concepts, Chapter 18.10.2

---

## Concurrency in Index Structures

Multiple threads may access and modify indexes simultaneously. We need mechanisms to ensure correctness while maximizing parallelism.

### Challenges

| Challenge | Description |
|-----------|-------------|
| **Concurrent reads** | Multiple threads reading same page |
| **Read-write conflicts** | Reader sees partial modification |
| **Write-write conflicts** | Concurrent modifications corrupt structure |
| **Structural changes** | Splits, merges during traversal |

---

## Latch Coupling (Crabbing)

### Basic Idea

Hold latch on parent while latching child, then release parent after child is safe.

```
Traversal:
1. Latch root (read mode for lookup, write mode for modification)
2. Find appropriate child
3. Latch child
4. If child is "safe" (won't split/merge), release parent latch
5. Move to child, repeat
```

### Safe vs Unsafe Nodes

| Operation | Safe Condition |
|-----------|----------------|
| **Insertion** | Node is not full (won't split) |
| **Deletion** | Node is more than half full (won't merge) |

### Crabbing Algorithm for Lookup

```
FUNCTION Lookup(key):
    node = root
    latch(node, READ)
    
    while node is not leaf:
        child = FindChild(node, key)
        latch(child, READ)
        unlatch(node)
        node = child
    
    # Search in leaf
    result = SearchInNode(node, key)
    unlatch(node)
    return result
```

### Crabbing Algorithm for Insertion

```
FUNCTION Insert(key, value):
    node = root
    latch(node, WRITE)
    
    while node is not leaf:
        child = FindChild(node, key)
        
        if IsSafeForInsert(child):
            # Child won't split, can release ancestors
            ReleaseAll ancestor latches
        
        latch(child, WRITE)
        node = child
    
    # Insert into leaf
    InsertIntoLeaf(node, key, value)
    
    if node overflows:
        SplitAndPropagate(node)
    
    unlatch(node)
```

### Example: Insertion with Split

```
Insert 75 into tree where node [60|70|80] is full:

1. Latch root (write)
2. Find child, check if safe (it will split - unsafe)
3. Latch child (write), keep parent latched
4. Reach leaf [60|70|80], latch it (write)
5. Insert causes split
6. Need to insert separator into parent
7. Parent is already latched - safe!
8. Split propagates up as needed
```

---

## B-link Trees

A **B-link tree** adds sibling pointers to B+ trees for better concurrency.

### Structure

```
        [50|100]
       /   |    \
    [25]--[75]--[125]
     |      |      |
   [leaf] [leaf] [leaf]
   
Each node has:
- High key: Maximum key in subtree
- Right sibling pointer
```

### Advantages

1. **Can release parent latch earlier**
   - After latching child, can immediately release parent
   - If child was split, use sibling pointer to find correct node

2. **Handle concurrent splits gracefully**
   - Split creates new sibling
   - Update sibling pointer
   - No need to hold parent during entire operation

### Concurrent Split Handling

```
Thread A inserting into node X:
1. Latch X (write)
2. If X is full:
   a. Allocate new node Y
   b. Move half of X's entries to Y
   c. Set Y's sibling pointer to X's old sibling
   d. Set X's sibling pointer to Y
   e. Insert separator into parent (may need to latch)
3. Unlatch X

Thread B traversing:
- If B sees X before split completes: normal traversal
- If B sees X after split, needs to go to Y:
  - Follows sibling pointer from X
  - High key tells B when to follow sibling
```

---

## Optimistic Latching

### Idea

Assume no structural changes will occur, verify after.

```
Optimistic Lookup:
1. Traverse without any latches (or with read latches briefly)
2. At leaf, verify we reached correct node
3. If verification fails (node was split), retry with pessimistic approach

Optimistic Insert:
1. Optimistically traverse to leaf
2. Latch leaf (write)
3. If leaf is safe, insert and done
4. If leaf is unsafe, release all, restart with crabbing
```

### Trade-offs

| Approach | Best For | Overhead |
|----------|----------|----------|
| **Pessimistic (Crabbing)** | High contention, many modifications | Higher latch overhead |
| **Optimistic** | Low contention, mostly reads | May need retries |

---

## Epoch-Based Reclamation

For lock-free data structures.

### Concept

1. **Threads announce their activity** (enter epoch)
2. **Safe to free memory** when all threads have exited epoch
3. **Used in Bw-Trees** and other lock-free indexes

```
Epoch-Based Memory Reclamation:

Thread enters epoch:
  - Increment active thread count for current epoch

Thread exits epoch:
  - Decrement active thread count

Garbage collection:
  - When epoch E has 0 active threads
  - All memory allocated in epochs ≤ E can be freed
```

### Bw-Tree (Microsoft Hekaton)

Lock-free B+ tree using epoch-based reclamation:

```
Key Techniques:
1. Mapping table: Page ID → Physical location
2. Delta records: Append modifications instead of in-place updates
3. Consolidation: Periodically apply deltas to create new base page
4. Epoch-based: Safe memory reclamation
```

---

## Summary

| Technique | Mechanism | Best For |
|-----------|-----------|----------|
| **Latch Coupling** | Hold parent while latching child | Traditional B+ trees |
| **B-link Trees** | Sibling pointers, high keys | High concurrency |
| **Optimistic Latching** | Verify after traversal | Read-heavy workloads |
| **Epoch-Based** | Lock-free with safe reclamation | Lock-free structures |

---

## Key Takeaways

1. **Latch Coupling**: Conservative approach, always safe
2. **B-link Trees**: Enable earlier latch release, better concurrency
3. **Optimistic Approaches**: Lower overhead when conflicts are rare
4. **Epoch-Based Reclamation**: Enables truly lock-free structures
5. **Trade-offs**: More concurrency often means more complexity

---

*Next Lecture: Sorting & Aggregation Algorithms*
