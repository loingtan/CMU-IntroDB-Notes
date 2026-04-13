# Lecture 04: Memory Management

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** September 8, 2025  
**Readings:** Database System Concepts, Chapter 13.2-13.5  
**Project:** Buffer Pool Manager  
**[📹 Watch Video](https://www.youtube.com/watch?v=8-2yv4z0VZc&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=4)**

---

## The Buffer Pool

The **buffer pool** is a region of main memory reserved by the DBMS for caching disk pages. It acts as an intermediary between the disk and the query execution engine.

### Why Buffer Pools Matter

| Factor | Impact |
|--------|--------|
| **Disk I/O is slow** | ~10ms for HDD, ~100μs for SSD |
| **Memory is fast** | ~100ns access time |
| **Data locality** | Recently accessed data likely to be accessed again |

**Performance gain**: 100,000x to 1,000,000x faster access for cached pages!

### Buffer Pool Architecture

```
+------------------+
|   Query Engine   |
+------------------+
         |
         v
+------------------+
|   Buffer Pool    |  ← Main Memory
| +--------------+ |
| | Frame 0      | | ← Page 42 (pinned)
| +--------------+ |
| | Frame 1      | | ← Page 7 (unpinned)
| +--------------+ |
| | Frame 2      | | ← Page 23 (pinned, dirty)
| +--------------+ |
| | ...          | |
| +--------------+ |
+------------------+
         |
         v
+------------------+
|      Disk        |
+------------------+
```

---

## Page Table

The **page table** maps page IDs to their locations in the buffer pool.

### Page Table Entry

```c
struct PageTableEntry {
    frame_id_t frame_id;      // Location in buffer pool
    int pin_count;            // Number of threads using this page
    bool is_dirty;            // Has the page been modified?
    std::shared_mutex latch;  // For thread-safe access
};
```

### Page Table Operations

**Lookup:**
```
Input: page_id
1. Hash page_id to find bucket
2. Search bucket for entry
3. Return frame_id if found, else NOT_FOUND
```

**Insert:**
```
Input: page_id, frame_id
1. Create new entry
2. Set pin_count = 1, is_dirty = false
3. Add to hash table
```

**Remove:**
```
Input: page_id
1. Verify pin_count == 0
2. Remove from hash table
```

---

## Frame Structure

Each frame in the buffer pool contains:

```
+------------------+
| Page ID          |  ← Identifies which page is stored
+------------------+
| Pin Count        |  ← Number of threads using this page
+------------------+
| Dirty Flag       |  ← True if page has been modified
+------------------+
| Latch            |  ← Synchronization primitive
+------------------+
| Page Data        |  ← Actual page content (4KB-16KB)
+------------------+
```

### Frame States

| State | Pin Count | Dirty | Description |
|-------|-----------|-------|-------------|
| **Empty** | 0 | false | Available for use |
| **Clean** | >0 | false | In use, unmodified |
| **Dirty** | >0 | true | In use, modified |
| **Evictable** | 0 | false | Can be replaced |

---

## Buffer Pool Operations

### Pinning a Page

When a thread needs to access a page:

```
FUNCTION FetchPage(page_id):
    1. Check page table for page_id
    
    2. IF page is in buffer pool:
        a. Increment pin_count
        b. Return pointer to page data
    
    3. IF page is NOT in buffer pool:
        a. Call replacement policy to find victim frame
        b. IF victim is dirty:
            - Write victim page to disk
        c. Read requested page from disk into victim frame
        d. Update page table
        e. Set pin_count = 1
        f. Return pointer to page data
```

### Unpinning a Page

When a thread is done with a page:

```
FUNCTION UnpinPage(page_id, is_dirty):
    1. Find page in page table
    
    2. Decrement pin_count
    
    3. IF is_dirty is true:
        - Set dirty flag in page table entry
    
    4. IF pin_count reaches 0:
        - Page becomes eligible for eviction
```

### Flushing a Page

Write a dirty page to disk:

```
FUNCTION FlushPage(page_id):
    1. Find page in page table
    
    2. Write page data to disk
    
    3. Clear dirty flag
    
    4. (Optional) Can keep page in memory or evict)
```

### New Page Allocation

```
FUNCTION NewPage():
    1. Find a free frame (or evict one)
    
    2. Assign new page ID
    
    3. Initialize page header
    
    4. Add to page table with pin_count = 1
    
    5. Return page ID and pointer to page data
```

---

## Page Replacement Policies

When the buffer pool is full and a new page is needed, we must **evict** an existing page.

### Least Recently Used (LRU)

Evict the page that hasn't been accessed for the longest time.

**Implementation:**
- Maintain a linked list ordered by access time
- Most recent at head, least recent at tail
- On access: move page to head
- On eviction: remove from tail

```
Access Order: A → B → C → A → D

Linked List: [D] → [C] → [B] → [A] → (evict A)
              ↑                    ↑
            Most Recent       Least Recent
```

**Advantages:**
- Good for workloads with temporal locality
- Simple to implement

**Disadvantages:**
- Sequential scans can pollute cache
- Overhead of moving elements

### Clock (Second Chance)

Approximation of LRU using a circular buffer.

**Algorithm:**
```
1. Each page has a reference bit (initially 0)
2. On access: set reference bit to 1
3. When looking for victim:
   a. If reference bit is 1: clear it, move to next
   b. If reference bit is 0: evict this page
```

**Visualization:**
```
    [A:1] → [B:0] → [C:1] → [D:0]
      ↑                       |
      └-----------------------┘
      
Clock hand moves, gives second chance to A and C
Evicts B (first 0 encountered)
```

**Advantages:**
- More efficient than LRU (no list manipulation)
- Similar performance to LRU

### Most Recently Used (MRU)

Evict the most recently used page.

**Use Case:** Sequential scans where pages won't be reused.

```
Query: SELECT * FROM large_table

Access pattern: Page 1 → Page 2 → Page 3 → ...

LRU would keep all pages (bad!)
MRU evicts just-accessed page (good!)
```

### Random

Pick a random page to evict.

**Advantages:**
- Very simple, low overhead
- No tracking needed
- Surprisingly effective in some workloads

---

## Buffer Pool Optimizations

### Multiple Buffer Pools

Separate pools for different purposes:

| Pool | Purpose | Size |
|------|---------|------|
| **Data Pool** | Table data | 70% |
| **Index Pool** | Index pages | 20% |
| **Temp Pool** | Temporary data | 10% |

**Benefits:**
- Each pool can have its own replacement policy
- Reduces contention
- Improves cache locality

### Pre-fetching

Anticipate future page accesses:

**Sequential Pre-fetching:**
```
When reading page N, also read pages N+1, N+2, ..., N+k
```

**Index Pre-fetching:**
```
When scanning index leaf pages, pre-fetch next leaf
```

**Benefits:**
- Overlaps I/O with computation
- Reduces total query time

### Scan Sharing

Multiple queries can share the same scan:

```
Query 1: SELECT * FROM R WHERE a > 5
Query 2: SELECT * FROM R WHERE a > 10

Both queries scan same range of R
→ Share the scan, filter results separately
```

**Benefits:**
- Reduces I/O when queries overlap
- Improves throughput

### Buffer Pool Bypass

For large sequential scans, bypass the buffer pool entirely:

```
IF scan_size > threshold:
    Use direct I/O (bypass buffer pool)
ELSE:
    Use normal buffer pool access
```

**Benefits:**
- Avoids polluting cache with one-time data
- Leaves buffer pool for random access patterns

---

## Dirty Page Management

### Write-Back Policy

Modified pages are not immediately written to disk.

**When to Write:**
1. **Eviction**: Victim page is dirty → write to disk
2. **Checkpoint**: Periodic flush of all dirty pages
3. **Commit**: For WAL protocols, log is flushed

### Force vs. No-Force

| Policy | Description | Systems |
|--------|-------------|---------|
| **Force** | All dirty pages written at commit | Rare (too slow) |
| **No-Force** | Dirty pages can stay in memory | Most systems |

**No-Force Benefits:**
- Multiple updates batched into single write
- Pages may be modified again before eviction

### Steal vs. No-Steal

| Policy | Description | Systems |
|--------|-------------|---------|
| **Steal** | Uncommitted data can be written to disk | Most systems |
| **No-Steal** | Uncommitted data stays in memory | Some systems |

**Steal Benefits:**
- Can evict pages from long-running transactions
- Better buffer pool utilization

**Steal Challenge:**
- Must be able to undo uncommitted changes (using WAL)

---

## Summary

| Concept | Description |
|---------|-------------|
| **Buffer Pool** | Memory cache for disk pages |
| **Page Table** | Maps page IDs to frame locations |
| **Pin Count** | Tracks how many threads use a page |
| **Dirty Flag** | Indicates page has been modified |
| **LRU** | Evict least recently used page |
| **Clock** | Efficient LRU approximation |
| **Pre-fetching** | Anticipate and load future pages |

---

## Key Takeaways

1. **Buffer Pool Bridges Speed Gap**: Caches disk pages in memory
2. **Page Table Manages Mapping**: Tracks which pages are in memory
3. **Pin Count Prevents Eviction**: Pages in use cannot be replaced
4. **Replacement Policies Matter**: Choose based on workload
5. **Optimizations Improve Performance**: Pre-fetching, scan sharing, bypass

---

*Next Lecture: Database Storage II - Alternative Storage Architectures*
