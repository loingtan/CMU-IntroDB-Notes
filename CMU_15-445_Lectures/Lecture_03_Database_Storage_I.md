# Lecture 03: Database Storage I

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** September 3, 2025  
**⚠️ No In-Class Lecture (Pre-recorded)**  
**Readings:** Database System Concepts, Chapter 12.1-12.4, 13.2-13.3  
**[📹 Watch Video](https://www.youtube.com/watch?v=PRLXdIMJhOg&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=3)**

---

## The Storage Hierarchy

Database systems must work within a hierarchy of storage technologies, each with different characteristics:

| Level | Technology | Latency | Cost/GB | Volatility |
|-------|-----------|---------|---------|------------|
| CPU Registers | SRAM | < 1 ns | Highest | Volatile |
| CPU Cache (L1/L2/L3) | SRAM | ~1-10 ns | Very High | Volatile |
| Main Memory | DRAM | ~100 ns | High | Volatile |
| SSD (NVMe) | Flash | ~10-100 μs | Medium | Non-volatile |
| SSD (SATA) | Flash | ~100 μs | Medium | Non-volatile |
| HDD | Magnetic | ~10 ms | Low | Non-volatile |
| Network Storage | Various | ~10-100 ms | Low | Non-volatile |
| Tape | Magnetic | Seconds | Very Low | Non-volatile |

### Key Observations

1. **100,000x gap** between memory and SSD access times
2. **100x gap** between SSD and HDD access times
3. **Volatile vs. Non-volatile** trade-off
4. Database systems must optimize for this hierarchy

---

## Why Disk-Oriented DBMS?

Most database systems are **disk-oriented** because:

1. **Databases are typically larger than available memory**
   - Enterprise databases: terabytes to petabytes
   - Server memory: gigabytes to terabytes

2. **Must survive system crashes (durability requirement)**
   - Volatile memory loses data on power failure
   - Non-volatile storage ensures persistence

3. **Need to handle datasets that grow over time**
   - Storage can be added incrementally
   - Memory expansion is more limited

### Design Goal

Make the database system work efficiently even though:
- Most data lives on disk
- Only a fraction fits in memory at once
- Disk I/O is orders of magnitude slower than memory access

---

## Database File Organization

### File Structure

- A database is stored as **one or more files** on disk
- The OS treats these as unstructured byte sequences
- The DBMS manages the mapping between logical data and physical storage

### File Types

| File Type | Purpose |
|-----------|---------|
| **Heap File** | Unordered storage of table data |
| **Index File** | B+ tree, hash table structures |
| **Log File** | Write-ahead log for recovery |
| **Catalog File** | Metadata about tables, indexes |
| **Temp File** | Temporary data during query processing |

### File Storage Managers

**Approach 1: OS File System**
- Use regular files managed by OS
- Simple, portable
- Limited control over placement

**Approach 2: Raw Disk (Direct I/O)**
- Bypass OS file system
- Direct control over block placement
- Better performance for sequential access

**Approach 3: Custom Storage Engine**
- Manage disk directly
- Full control over layout
- Complex to implement

---

## Page (Block) Concept

The **page** (or block) is the fundamental unit of I/O in database systems.

### Page Characteristics

| Property | Typical Value |
|----------|---------------|
| Size | 4KB, 8KB, or 16KB |
| Alignment | Page-aligned in memory and on disk |
| Atomicity | Read/write entire page at once |

### Why Fixed-Size Pages?

1. **Simplifies memory management**
2. **Enables efficient buffer pool**
3. **Matches hardware characteristics** (disk sectors, memory pages)
4. **Facilitates direct I/O**

### Page Types

| Type | Description |
|------|-------------|
| **Data Pages** | Store actual table data |
| **Index Pages** | Store index structures (B+ trees, hash tables) |
| **System Pages** | Store metadata, catalogs, statistics |
| **Overflow Pages** | Store data that doesn't fit in regular pages |

---

## Page Layout

### Slotted Page Structure

The most common page layout for variable-length records:

```
+------------------+  ← Start of Page (lower addresses)
| Header           |  ← Page metadata
+------------------+
| Slot Array       |  ← Directory of record locations
+------------------+
|                  |
| Free Space       |  ← Grows toward end of page
|                  |
+------------------+
| Record N         |  ← Records grow from end toward start
+------------------+
| Record N-1       |
+------------------+
| ...              |
+------------------+
| Record 1         |
+------------------+  ← End of Page (higher addresses)
```

### Page Header Contents

```c
struct PageHeader {
    uint32_t page_id;           // Unique page identifier
    uint32_t free_space_offset; // Start of free space
    uint16_t num_slots;         // Number of slots in use
    uint16_t free_space;        // Bytes of free space remaining
    uint32_t next_page;         // For linked list of pages
    uint32_t prev_page;         // For doubly-linked list
    // ... other metadata
};
```

### Slot Array

The slot array acts as a directory:

```
Slot 0: [offset=800, length=50]  → Record at byte 800, 50 bytes
Slot 1: [offset=750, length=30]  → Record at byte 750, 30 bytes
Slot 2: [offset=0, length=0]     → Deleted record (empty slot)
Slot 3: [offset=700, length=40]  → Record at byte 700, 40 bytes
```

**Key Benefits:**
- Records can be **moved within a page** without updating external references
- **Variable-length records** are supported efficiently
- **Handles fragmentation** gracefully (compaction possible)

---

## Record Layout

### Fixed-Length Records

All records have the same size:

```
[Record 1: 100 bytes][Record 2: 100 bytes][Record 3: 100 bytes]...
```

**Address Calculation:**
```
address = page_start + header_size + (record_size * slot_number)
```

**Advantages:**
- Simple offset calculation
- No indirection needed
- Fast access

**Disadvantages:**
- Wastes space for sparse data
- Cannot handle variable-length fields

### Variable-Length Records

Records have different sizes:

```
[Record 1: 45 bytes]...[Record 2: 120 bytes]...[Record 3: 30 bytes]...
```

**Address Calculation:**
```
1. Look up slot in slot array
2. Get offset and length from slot
3. address = page_start + offset
```

**Advantages:**
- Flexible, handles any data
- Space efficient

**Disadvantages:**
- Requires indirection (slot lookup)
- More complex management

### Record Header

Every record typically contains:

```c
struct RecordHeader {
    // Visibility information (for MVCC)
    txn_id_t created_by;    // Transaction that created this version
    txn_id_t deleted_by;    // Transaction that deleted this version
    
    // Null bitmap
    uint8_t null_bitmap[];  // One bit per nullable column
    
    // Variable-length offsets
    uint16_t var_offsets[]; // Offsets to variable-length fields
    
    // Actual data follows
};
```

---

## Data Representation

### Primitive Types

| Type | Size | Notes |
|------|------|-------|
| BOOLEAN | 1 byte | 0 = false, 1 = true |
| TINYINT | 1 byte | -128 to 127 |
| SMALLINT | 2 bytes | -32,768 to 32,767 |
| INTEGER/INT | 4 bytes | ~±2 billion |
| BIGINT | 8 bytes | ~±9 quintillion |
| FLOAT/REAL | 4 bytes | IEEE 754 single precision |
| DOUBLE | 8 bytes | IEEE 754 double precision |
| DECIMAL(p,s) | Variable | Fixed-point arithmetic |
| DATE | 4 bytes | Days since epoch (e.g., 2000-01-01) |
| TIME | 4 bytes | Microseconds since midnight |
| TIMESTAMP | 8 bytes | Date + time combined |

### Variable-Length Data

**VARCHAR(n):**
```
[length: 2 bytes][data: variable]
```

**TEXT/BLOB:**
- Small values: Inline in page
- Large values: Overflow pages (TOAST in PostgreSQL)
- Very large: External files with references

**Overflow Page Handling:**
```
Page 1: [header][data...][pointer to overflow]
Overflow Page: [header][continuation of data][pointer or end]
```

---

## System Catalog

The DBMS maintains metadata about the database in the **system catalog**:

### Catalog Contents

| Information | Description |
|-------------|-------------|
| **Table Schemas** | Column names, types, constraints |
| **Indexes** | Index types, columns, structures |
| **Views** | View definitions |
| **Users & Permissions** | Access control information |
| **Statistics** | Table sizes, column distributions |
| **Storage Info** | Page counts, file locations |

### Example Catalog Tables

```sql
-- PostgreSQL-style catalog
SELECT * FROM pg_tables;        -- All tables
SELECT * FROM pg_indexes;       -- All indexes
SELECT * FROM pg_attribute;     -- Column information
SELECT * FROM pg_stats;         -- Column statistics
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Storage Hierarchy** | Memory is fast but small; disk is slow but large |
| **Page** | Fundamental I/O unit (4KB-16KB) |
| **Slotted Page** | Flexible layout for variable-length records |
| **Slot Array** | Directory for record locations within page |
| **Fixed-Length Records** | Simple but inflexible |
| **Variable-Length Records** | Flexible with slot indirection |
| **System Catalog** | Metadata about the database |

---

## Key Takeaways

1. **Disk-Oriented Design**: Databases are designed for data larger than memory
2. **Page-Based I/O**: All disk operations work in page-sized units
3. **Slotted Pages**: Enable efficient variable-length record management
4. **Indirection**: Slot array allows record movement without pointer updates
5. **Catalog**: Stores metadata needed for query processing

---

*Next Lecture: Memory Management - The Buffer Pool Manager*
