# Lecture 07: Hash Tables

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** September 17, 2025  
**Readings:** Database System Concepts, Chapter 14.5, 24.5  
**Flash Talk:** Karthik Ranganathan (YugabyteDB)  
**[📹 Watch Video](https://www.youtube.com/watch?v=nuNW8IfgPNU&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=7)**

---

## Hash Table Basics

A **hash table** provides O(1) average-case lookup, insertion, and deletion by using a hash function to map keys to array indices.

### Components

1. **Hash Function**: Maps keys to bucket indices
2. **Bucket Array**: Stores the actual data
3. **Collision Resolution**: Handles when two keys hash to the same bucket

### Basic Operations

| Operation | Average Case | Worst Case |
|-----------|--------------|------------|
| **Lookup** | O(1) | O(n) |
| **Insert** | O(1) | O(n) |
| **Delete** | O(1) | O(n) |

---

## Hash Functions for Databases

### Desirable Properties

| Property | Description |
|----------|-------------|
| **Deterministic** | Same key always produces same hash |
| **Uniform** | Distributes keys evenly across buckets |
| **Fast** | Computation should be quick |
| **Avalanche Effect** | Small key changes cause large hash changes |

### Common Hash Functions

| Function | Characteristics | Use Case |
|----------|-----------------|----------|
| **MurmurHash** | Fast, good distribution, widely used | General purpose |
| **CityHash** | Optimized for short strings | String keys |
| **xxHash** | Extremely fast | Checksums, fast lookups |
| **FNV-1a** | Simple, decent distribution | Simple implementations |
| **SHA-256** | Cryptographic, slow | Security applications |

### Example: MurmurHash

```cpp
// Pseudocode for MurmurHash
uint32_t MurmurHash(const void* key, int len, uint32_t seed) {
    // Mix bits from key into hash
    // Use bit shifts and multiplications
    // Finalize with additional mixing
    return hash;
}
```

---

## Static Hashing

### Chained Hashing

Each bucket contains a linked list of entries.

```
Bucket Array:
[0] → [A] → [B]
[1] → [C]
[2] → ∅
[3] → [D] → [E] → [F]

Hash(A) = 0, Hash(B) = 0, Hash(C) = 1
Hash(D) = 3, Hash(E) = 3, Hash(F) = 3
```

**Operations:**
```
Insert(key, value):
    bucket = hash(key) % num_buckets
    append (key, value) to bucket's linked list

Lookup(key):
    bucket = hash(key) % num_buckets
    for each entry in bucket's list:
        if entry.key == key:
            return entry.value
    return NOT_FOUND
```

**Advantages:**
- Simple to implement
- Handles any number of collisions

**Disadvantages:**
- Can degrade to O(n) with many collisions
- Poor cache locality (linked list traversal)

### Linear Probe Hashing (Open Addressing)

All entries stored in the bucket array itself.

```
Insert 5: hash(5) = 2 → Bucket[2] = 5 ✓
Insert 15: hash(15) = 2 → Bucket[2] occupied
           Try Bucket[3] → Bucket[3] = 15 ✓
Insert 25: hash(25) = 2 → Bucket[2] occupied
           Try Bucket[3] → occupied
           Try Bucket[4] → Bucket[4] = 25 ✓

Result: [ ][ ][5][15][25][ ]
```

**Operations:**
```
Insert(key, value):
    idx = hash(key) % num_buckets
    while bucket[idx] is occupied:
        idx = (idx + 1) % num_buckets
    bucket[idx] = (key, value)

Lookup(key):
    idx = hash(key) % num_buckets
    while bucket[idx] is not empty:
        if bucket[idx].key == key:
            return bucket[idx].value
        idx = (idx + 1) % num_buckets
    return NOT_FOUND
```

**Advantages:**
- Better cache locality (contiguous array)
- No pointer overhead

**Disadvantages:**
- Suffers from **clustering** (groups of occupied slots)
- Deletion is complex (need tombstones)

### Robin Hood Hashing

Variant of linear probing that reduces variance in probe lengths.

**Key Idea:** When inserting, "steal" from entries that are "richer" (closer to their ideal position).

```
Insert key with ideal position H:
    current = H
    distance = 0
    
    while bucket[current] is occupied:
        if bucket[current].distance_from_ideal < distance:
            // Current entry is "richer", swap
            swap(key, bucket[current])
            distance = bucket[current].distance_from_ideal
        
        current = (current + 1) % num_buckets
        distance++
    
    bucket[current] = key
    bucket[current].distance = distance
```

**Benefit:** Reduces worst-case probe length, more consistent performance.

---

## Dynamic Hashing

Static hash tables require knowing the number of elements in advance. Dynamic schemes adapt to data size.

### Extendible Hashing

Uses a directory of pointers to buckets.

```
Global Depth = 2

Directory (size = 2^2 = 4):
[00] → Bucket A: [4, 8, 12]
[01] → Bucket A: [4, 8, 12]
[10] → Bucket B: [2, 6]
[11] → Bucket B: [2, 6]

Bucket A (Local Depth = 1): stores keys where hash ends in 0
Bucket B (Local Depth = 1): stores keys where hash ends in 10 or 11
```

**When Bucket Overflows:**
1. If local depth < global depth:
   - Split bucket
   - Redistribute entries based on additional hash bit
   - Update directory pointers

2. If local depth = global depth:
   - Double directory size (global depth++)
   - Split bucket
   - Update directory

**Advantages:**
- Only splits overflowing bucket
- Directory grows gradually

**Disadvantages:**
- Directory can grow large
- Indirection through directory

### Linear Hashing

Grows one bucket at a time (not doubling).

```
Initial: 4 buckets (0, 1, 2, 3)
Split Pointer = 0

When load factor exceeds threshold:
    Split bucket at split pointer
    Create new bucket at end
    Redistribute entries
    Increment split pointer
    
If split pointer reaches current size:
    Double size
    Reset split pointer
```

**Advantages:**
- Simpler than extendible hashing
- Grows gradually

**Disadvantages:**
- May require overflow chains temporarily
- More complex hash function

---

## Hash Tables in Database Systems

### When to Use Hash Tables

| Use Case | Why Hash Tables? |
|----------|------------------|
| **Exact-match lookups** | O(1) access: `WHERE id = 123` |
| **Hash joins** | Efficient equijoins on large tables |
| **Hash aggregation** | GROUP BY with no index |
| **Duplicate elimination** | DISTINCT operations |

### When NOT to Use Hash Tables

| Use Case | Better Alternative |
|----------|-------------------|
| **Range queries** | B+ Tree: `WHERE id > 100` |
| **Prefix searches** | B+ Tree: `WHERE name LIKE 'John%'` |
| **Ordering** | B+ Tree: `ORDER BY` |
| **Nearest neighbor** | Specialized index (R-tree, etc.) |

---

## Hash Joins

Hash joins are one of the most efficient join algorithms.

### Basic Hash Join

```
Phase 1: Build
---------
For each row r in R (smaller table):
    hash_key = hash(r.join_column)
    insert r into hash_table[hash_key]

Phase 2: Probe
------------
For each row s in S (larger table):
    hash_key = hash(s.join_column)
    for each r in hash_table[hash_key]:
        if r.join_column == s.join_column:
            output (r, s)
```

**Cost:** 3 × (|R| + |S|) I/Os (read both, write hash table, probe)

### Grace Hash Join

When hash table doesn't fit in memory:

```
Phase 1: Partition
-----------------
For each row r in R:
    partition = hash(r.join_column) % N
    write r to R_partition[partition]

For each row s in S:
    partition = hash(s.join_column) % N
    write s to S_partition[partition]

Phase 2: Build & Probe
---------------------
For each partition i from 0 to N-1:
    Build hash table from R_partition[i]
    Probe with S_partition[i]
    Output matches
```

**Cost:** 3 × (|R| + |S|) I/Os (partition + read + write)

---

## Summary

| Hashing Scheme | Collision Handling | Dynamic | Use Case |
|----------------|-------------------|---------|----------|
| **Chained** | Linked lists | No | Simple implementations |
| **Linear Probe** | Open addressing | No | Cache-friendly |
| **Robin Hood** | Open addressing | No | Consistent performance |
| **Extendible** | Directory | Yes | In-memory indexes |
| **Linear** | Overflow chains | Yes | Disk-based indexes |

---

## Key Takeaways

1. **Hash Functions Matter**: Good distribution is critical for performance
2. **Collision Resolution**: Chaining vs. open addressing trade-offs
3. **Dynamic Hashing**: Extendible and linear for growing data
4. **Hash Joins**: Most efficient for large equijoins
5. **Use the Right Tool**: Hash tables for equality, B+ trees for ranges

---

*Next Lecture: Indexes & Filters I - B+ Trees and Tree Indexes*
