# Lecture 09: Indexes & Filters II

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** September 24, 2025  
**Readings:** Database System Concepts, Chapter 14.1-14.4, 24.1  
**Homework:** Indexes & Filters  
**Project:** Database Index

---

## Skip Lists

A **skip list** is a probabilistic alternative to balanced trees.

### Structure

```
Level 3:  [1]------------------------------------------>[NIL]
             ↓
Level 2:  [1]--------->[25]---------------------------->[NIL]
             ↓              ↓
Level 1:  [1]-->[10]-->[25]-->[35]--------------------->[NIL]
             ↓       ↓       ↓       ↓
Level 0:  [1]-->[10]-->[25]-->[35]-->[50]-->[60]-->[NIL]
             
Bottom level (Level 0): Linked list of all elements
Higher levels: Express lanes that skip elements
```

### Properties

| Property | Description |
|----------|-------------|
| **Probabilistic** | Randomized level assignment |
| **Expected O(log n)** | Search, insert, delete |
| **Simple** | No complex rebalancing |
| **Lock-free friendly** | Better concurrency than B+ trees |

### Level Assignment

```
Insert key:
    level = 0
    while random() < 0.5 and level < max_level:
        level++
    
    Create node with tower of height (level + 1)
    Link into lists at each level
```

**Expected number of levels:** O(log n)

### Operations

**Search:**
```
FUNCTION Search(key):
    node = head
    
    for level from max_level down to 0:
        while node.next[level].key < key:
            node = node.next[level]
    
    # Now at level 0
    node = node.next[0]
    
    if node.key == key:
        return node.value
    else:
        return NOT_FOUND
```

**Insertion:**
```
FUNCTION Insert(key, value):
    update[] = array of nodes to update at each level
    node = head
    
    # Find position at each level
    for level from max_level down to 0:
        while node.next[level].key < key:
            node = node.next[level]
        update[level] = node
    
    # Create new node with random level
    new_level = RandomLevel()
    new_node = CreateNode(key, value, new_level)
    
    # Link into all levels
    for level from 0 to new_level:
        new_node.next[level] = update[level].next[level]
        update[level].next[level] = new_node
```

### Skip Lists vs B+ Trees

| Aspect | Skip List | B+ Tree |
|--------|-----------|---------|
| Implementation | Simple | Complex |
| Rebalancing | None (probabilistic) | Explicit splits/merges |
| Cache efficiency | Moderate (pointer chasing) | Good (node-based) |
| Concurrency | Excellent (lock-free possible) | Good (with latches) |
| Range scans | Good | Excellent |
| Used in | MemTables, Redis | Disk-based indexes |

---

## Radix Trees (Tries)

A **radix tree** (or trie) stores keys by their bit/byte representation.

### Structure

```
Keys: "apple", "app", "bat", "bath"

                root
               /    \
             'a'    'b'
             /        \
         'p'          'a'
         /              \
     'p'               't'
     / \                 \
  'l'   [END]           'h'
  /                       \
'e'[apple]              [bath]
/
[END]
```

### Properties

| Property | Description |
|----------|-------------|
| **Key lookup time** | O(key_length), not tree height |
| **No hashing** | Direct key comparison |
| **Prefix support** | Natural prefix matching |
| **Compressed** | Path compression saves space |

### Compressed Radix Tree

Merge single-child nodes:

```
Uncompressed: a → p → p → l → e
Compressed: "apple"

                root
               /    \
           "app"    "ba"
           /   \      \
        "le"   [END]  "th"
         /              \
     [apple]           [bath]
```

### Use Cases

| Application | Why Radix Tree? |
|-------------|-----------------|
| **IP routing tables** | Longest prefix match |
| **String dictionaries** | Prefix searches |
| **Genome indexing** | DNA sequence matching |
| **Auto-complete** | Prefix-based suggestions |

---

## Bloom Filters

A **Bloom filter** is a space-efficient probabilistic data structure for membership testing.

### Properties

| Property | Description |
|----------|-------------|
| **Space efficient** | Much smaller than storing all keys |
| **No false negatives** | If "not in set", definitely not in set |
| **Possible false positives** | If "in set", might not be in set |
| **Cannot delete** | Without extensions |

### Structure

```
Bit array of size m:
[0|0|0|0|0|0|0|0|0|0]  (m=10)

k hash functions: h1, h2, ..., hk
```

### Operations

**Insert element x:**
```
for i from 1 to k:
    index = hi(x) % m
    bit_array[index] = 1
```

**Query element x:**
```
for i from 1 to k:
    index = hi(x) % m
    if bit_array[index] == 0:
        return DEFINITELY_NOT_IN_SET

return PROBABLY_IN_SET  (may be false positive)
```

### Example

```
Insert "hello":
  h1("hello") % 10 = 2  → set bit 2
  h2("hello") % 10 = 5  → set bit 5
  h3("hello") % 10 = 8  → set bit 8

Bit array: [0|0|1|0|0|1|0|0|1|0]

Query "world":
  h1("world") % 10 = 2  → bit 2 is 1 ✓
  h2("world") % 10 = 4  → bit 4 is 0 ✗
  
Result: DEFINITELY_NOT_IN_SET ✓

Query "maybe":
  h1("maybe") % 10 = 2  → bit 2 is 1 ✓
  h2("maybe") % 10 = 5  → bit 5 is 1 ✓
  h3("maybe") % 10 = 8  → bit 8 is 1 ✓
  
Result: PROBABLY_IN_SET (may be false positive)
```

### False Positive Rate

```
False Positive Rate ≈ (1 - e^(-kn/m))^k

Where:
  n = number of elements inserted
  m = bit array size
  k = number of hash functions

Optimal k = (m/n) × ln(2)
```

### Bloom Filters in Databases

| Use Case | How It Helps |
|----------|--------------|
| **LSM-Tree SSTables** | Skip SSTables that don't contain key |
| **Query Optimization** | Avoid unnecessary index lookups |
| **Distributed Systems** | Reduce network requests |
| **Join Optimization** | Bloom join filter |

**LSM-Tree Example:**
```
100 SSTables, looking for key "foo"

Without Bloom filter:
  Check all 100 SSTables

With Bloom filter per SSTable:
  Check 100 Bloom filters (in memory, fast)
  Only check SSTables where filter says "maybe"
  → Maybe check 5 SSTables instead of 100
```

---

## Index Selection Guidelines

### When to Create an Index

| Scenario | Recommendation |
|----------|----------------|
| **Frequent lookups on column** | Create index |
| **Foreign key columns** | Create index (for joins) |
| **Range queries** | B+ tree index |
| **Equality queries only** | Hash or B+ tree |
| **Low cardinality** | Consider bitmap index |
| **Text search** | Full-text index |

### When NOT to Create an Index

| Scenario | Reason |
|----------|--------|
| **Small tables** | Sequential scan is faster |
| **Frequent updates** | Index maintenance overhead |
| **Low selectivity queries** | Won't use index anyway |
| **Columns with many NULLs** | Wasted space |

### Covering Indexes

An index that contains all columns needed for a query.

```sql
-- Query
SELECT first_name, last_name FROM employees WHERE employee_id = 123;

-- Covering index
CREATE INDEX idx_cover ON employees(employee_id, first_name, last_name);

-- Index-only scan: No need to access table!
```

---

## Summary

| Structure | Best For | Key Property |
|-----------|----------|--------------|
| **B+ Tree** | Range queries, general purpose | Self-balancing |
| **Skip List** | In-memory, concurrent access | Probabilistic balance |
| **Radix Tree** | Prefix matching, IP routing | Key-length lookup |
| **Bloom Filter** | Membership testing, filtering | Space-efficient, probabilistic |

---

## Key Takeaways

1. **Skip Lists**: Simple, concurrent-friendly alternative to B+ trees
2. **Radix Trees**: Excellent for prefix-based operations
3. **Bloom Filters**: Cheap way to avoid unnecessary lookups
4. **Choose Index Type Based on Query Pattern**: No universal best index
5. **Covering Indexes**: Can eliminate table access entirely

---

*Next Lecture: Index Concurrency Control - Managing Concurrent Access*
