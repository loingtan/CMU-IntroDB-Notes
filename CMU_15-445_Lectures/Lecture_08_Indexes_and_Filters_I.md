# Lecture 08: Indexes & Filters I

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** September 22, 2025  
**Readings:** Database System Concepts, Chapter 14.1-14.4

---

## Introduction to Indexes

An **index** is a data structure that speeds up data retrieval operations on a database table. Think of it like the index in a book—it helps you find information quickly without scanning every page.

### Why Indexes Matter

| Without Index | With Index |
|---------------|------------|
| Sequential scan: O(n) | Tree lookup: O(log n) |
| 1M rows = scan all | 1M rows = ~20 comparisons |

### Index Trade-offs

| Benefit | Cost |
|---------|------|
| Faster reads | Slower writes (update index) |
| Faster sorts | More storage space |
| Faster joins | Maintenance overhead |

---

## B+ Trees

The **B+ Tree** is the most widely used index structure in database systems.

### B+ Tree Properties

| Property | Description |
|----------|-------------|
| **Self-balancing** | All leaf nodes at same depth |
| **Order M** | Each node holds M/2 to M keys |
| **All data in leaves** | Internal nodes only for navigation |
| **Leaf nodes linked** | Efficient range scans |
| **Sorted keys** | Enables range queries |

### B+ Tree Structure

```
                    [50|100]
                   /    |    \
            [10|25]  [60|75]  [120|150]
            /  |  \   /  |  \   /   |   \
        [1,5,10] [15,20,25] [50,55,60] [65,70,75] [100,110] [120,130] [150,160,170]
        
Internal Nodes: Contain keys for navigation
Leaf Nodes: Contain actual data (or pointers to data)
```

### Node Structure

**Internal Node:**
```
+------------------+
| Header           |  ← Key count, is_leaf flag
+------------------+
| Key Array        |  ← Sorted keys: [k1, k2, ..., kn]
+------------------+
| Child Pointers   |  ← [p0, p1, p2, ..., pn]
+------------------+

Navigation: keys in [p0] < k1 ≤ keys in [p1] < k2 ≤ ...
```

**Leaf Node:**
```
+------------------+
| Header           |  ← Key count, is_leaf flag
+------------------+
| Key Array        |  ← Sorted keys
+------------------+
| Value Array      |  ← Record IDs or actual data
+------------------+
| Next Pointer     |  → Pointer to next leaf
+------------------+
| Prev Pointer     |  → Pointer to previous leaf
+------------------+
```

### B+ Tree Operations

#### Lookup

```
FUNCTION Lookup(key):
    node = root
    
    while node is not leaf:
        # Find child pointer
        i = 0
        while i < node.num_keys and key >= node.keys[i]:
            i++
        node = node.children[i]
    
    # Search leaf
    for i from 0 to node.num_keys:
        if node.keys[i] == key:
            return node.values[i]
    
    return NOT_FOUND
```

**Cost:** O(log_M N) where M is order, N is number of keys

#### Insertion

```
FUNCTION Insert(key, value):
    leaf = FindLeaf(key)
    
    if leaf has space:
        Insert key/value in sorted order
    else:
        # Leaf is full, need to split
        new_leaf = AllocateNode()
        
        # Distribute keys between leaf and new_leaf
        mid = leaf.num_keys / 2
        new_leaf.keys = leaf.keys[mid:]
        new_leaf.values = leaf.values[mid:]
        leaf.keys = leaf.keys[:mid]
        leaf.values = leaf.values[:mid]
        
        # Link leaves
        new_leaf.next = leaf.next
        leaf.next = new_leaf
        
        # Insert separator key into parent
        InsertIntoParent(leaf, new_leaf.keys[0], new_leaf)
```

**Splitting Internal Nodes:**
```
When internal node is full:
    1. Create new internal node
    2. Move half the keys/pointers to new node
    3. Promote middle key to parent
    4. If parent is full, split recursively
    5. If root splits, create new root
```

#### Deletion

```
FUNCTION Delete(key):
    leaf = FindLeaf(key)
    
    if key not in leaf:
        return NOT_FOUND
    
    Remove key/value from leaf
    
    if leaf is underfull (keys < M/2):
        # Try to redistribute from sibling
        if sibling has extra keys:
            Redistribute(leaf, sibling)
        else:
            # Merge with sibling
            Merge(leaf, sibling)
            Remove separator key from parent
            
            if parent is underfull:
                HandleUnderfull(parent)  # Recursive
```

### B+ Tree vs B-Tree

| Feature | B-Tree | B+ Tree |
|---------|--------|---------|
| Data location | Internal + leaf nodes | Leaf nodes only |
| Leaf linkage | No | Yes (linked list) |
| Range scans | Inefficient | Efficient |
| Key duplication | No | Yes (copies in leaves) |
| Fanout | Lower | Higher (more keys per node) |

**Modern databases use B+ Trees almost exclusively.**

---

## B+ Tree Optimizations

### Prefix Compression

Store common prefix once per node.

```
Keys in node: ["database", "datastructure", "datatype"]
Common prefix: "data"
Stored as: prefix="data" + ["base", "structure", "type"]

Space saved: ~40% for these keys
```

### Suffix Truncation

Internal node keys only need to be separators.

```
Original keys: ["apple", "banana", "cherry"]
Separator between apple and banana: "b" is sufficient

Can truncate: "banana" → "b"
```

### Bulk Loading

Efficiently build B+ tree from sorted data.

```
Algorithm:
1. Sort all key-value pairs
2. Fill leaf nodes completely (no splits)
3. Build internal nodes level by level
4. Much faster than individual insertions
```

**When to use:**
- Initial data load
- Index rebuild
- Bulk import operations

### Pointer Swizzling

Convert page IDs to memory pointers when page is pinned.

```
Without swizzling:
  Follow pointer: Page ID → Page Table → Frame → Page

With swizzling:
  Follow pointer: Memory Pointer → Page (direct!)
  
Benefit: Avoids page table lookup during traversal
```

---

## Clustered vs. Non-Clustered Indexes

### Clustered Index

Determines the physical order of data in the table.

```
Table: Employees
Clustered Index: EmployeeID

Physical storage order: 1, 2, 3, 4, 5, ...
```

**Properties:**
- Only one clustered index per table
- Data is physically sorted by key
- Leaf nodes contain actual row data

### Non-Clustered Index

Separate structure with pointers to data.

```
Index on: LastName

Index entries: ["Alice"] → RowID: 5
               ["Bob"] → RowID: 2
               ["Carol"] → RowID: 8
```

**Properties:**
- Multiple non-clustered indexes per table
- Leaf nodes contain (key, pointer) pairs
- Pointer = RowID or clustered index key

---

## Composite Indexes

Index on multiple columns.

```sql
CREATE INDEX idx_name ON employees(last_name, first_name);
```

**Key Order Matters!**

```
Index structure: ["Smith", "Alice"] → RowID
                 ["Smith", "Bob"] → RowID
                 ["Smith", "Carol"] → RowID
                 ["Jones", "Alice"] → RowID
```

**Can efficiently answer:**
- `WHERE last_name = 'Smith'`
- `WHERE last_name = 'Smith' AND first_name = 'Alice'`
- `WHERE last_name = 'Smith' ORDER BY first_name`

**Cannot efficiently answer:**
- `WHERE first_name = 'Alice'` (first column not specified)

### Index Matching Rules

```sql
-- Index: (a, b, c)

WHERE a = 1              -- Uses index ✓
WHERE a = 1 AND b = 2    -- Uses index ✓
WHERE a = 1 AND c = 3    -- Partial (only a) ⚠
WHERE b = 2              -- Cannot use index ✗
WHERE a > 1 AND b = 2    -- Range on a, can't use b ⚠
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **B+ Tree** | Self-balancing tree, all data in leaves |
| **Order M** | Node capacity, affects tree height |
| **Lookup** | O(log N) by traversing from root |
| **Insertion** | May cause splits, propagate up |
| **Deletion** | May cause merges or redistribution |
| **Clustered** | Determines physical order |
| **Composite** | Multiple columns, order matters |

---

## Key Takeaways

1. **B+ Trees are Ubiquitous**: Used in virtually all relational databases
2. **All Data in Leaves**: Enables efficient range scans
3. **Self-Balancing**: Guaranteed O(log N) operations
4. **Leaf Linking**: Sequential access is fast
5. **Composite Indexes**: Column order critically important

---

*Next Lecture: Indexes & Filters II - Advanced Index Structures*
