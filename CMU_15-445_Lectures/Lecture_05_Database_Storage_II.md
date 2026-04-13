# Lecture 05: Database Storage II

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** September 10, 2025  
**Readings:** Database System Concepts, Chapter 14.8.1, 24.2  
**Homework:** Storage  
**Flash Talk:** Joseph Victor (SingleStore)  
**[📹 Watch Video](https://www.youtube.com/watch?v=2_sTdS4h-bY&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=5)**

---

## Alternative Storage Architectures

While traditional databases use page-oriented heap storage, modern systems employ alternative architectures optimized for specific workloads.

---

## Log-Structured Storage

### LSM-Tree (Log-Structured Merge Tree)

Popularized by systems like **LevelDB**, **RocksDB**, **Cassandra**, and **HBase**.

### Architecture Overview

```
+-------------+
| MemTable    |  ← In-memory sorted structure (skip list or B+ tree)
| (Active)    |
+-------------+
      |
      | When full, flush to disk
      v
+-------------+
| SSTable L0  |  ← Immutable sorted files (newest)
+-------------+
      |
      | Compaction merges levels
      v
+-------------+
| SSTable L1  |  ← Larger files
+-------------+
      |
      v
+-------------+
| SSTable L2  |  ← Even larger, less frequently accessed
+-------------+
      |
      v
+-------------+
| SSTable L3  |  ← Largest, oldest data
+-------------+
```

### Write Path

```
1. Write to Write-Ahead Log (WAL) for durability
2. Insert into MemTable (in-memory sorted structure)
3. When MemTable fills:
   a. Freeze current MemTable (becomes immutable)
   b. Create new active MemTable
   c. Flush frozen MemTable to disk as SSTable (Level 0)
4. Background compaction merges SSTables
```

### Read Path

```
1. Check MemTable first (most recent data)
2. Check immutable MemTables (if any)
3. Check SSTables from newest (L0) to oldest (L3)
4. Use Bloom filters to skip SSTables that don't contain the key
5. Return first found value
```

### SSTable Format

```
+------------------+
| Data Blocks      |  ← Sorted key-value pairs
+------------------+
| Index Block      |  ← Sparse index of keys
+------------------+
| Bloom Filter     |  ← Probabilistic membership test
+------------------+
| Footer           |  ← Metadata
+------------------+
```

### Compaction

**Why Compaction?**
- Multiple versions of same key accumulate
- Read performance degrades with more SSTables
- Reclaim space from overwritten/deleted keys

**Level-Based Compaction (RocksDB):**
```
Level 0: Up to 4 files, overlapping key ranges
Level 1: 10MB total, non-overlapping
Level 2: 100MB total
Level 3: 1GB total
...

When level exceeds size threshold:
  - Pick file(s) to compact
  - Merge with overlapping files in next level
  - Write new merged files
```

**Size-Tiered Compaction (Cassandra):**
```
- When enough similar-sized SSTables accumulate
- Merge them into one larger SSTable
- Simpler but more space amplification
```

### LSM-Tree Trade-offs

| Aspect | Advantage | Disadvantage |
|--------|-----------|--------------|
| **Write Performance** | Sequential writes are extremely fast | Compaction causes write amplification |
| **Read Performance** | Good for recent data | May check multiple SSTables |
| **Space Efficiency** | Compression-friendly | Space amplification during compaction |
| **Range Queries** | Efficient (sorted data) | Slower than B+ trees |
| **Point Lookups** | Bloom filters help | Still slower than B+ trees |

---

## Index-Organized Storage

### Clustered Index (Index-Organized Table)

In an **Index-Organized Table (IOT)**, the table data is stored directly in the index structure.

**Traditional Heap + Index:**
```
Index: [key → row_id]
Heap:  [row_id → full row data]

Lookup: Index → row_id → Heap access
```

**Index-Organized Table:**
```
Clustered Index: [key → full row data]

Lookup: Index → full row (no heap access!)
```

### MySQL InnoDB Example

```sql
-- Primary key is the clustered index
CREATE TABLE employee (
    id INT PRIMARY KEY,  -- Clustered index
    name VARCHAR(50),
    salary DECIMAL(10,2)
);

-- Data is physically ordered by id
-- Secondary indexes point to primary key
```

**Secondary Index in InnoDB:**
```
Secondary Index: [name → primary_key_value]
Then lookup primary key in clustered index
```

### Trade-offs

| Aspect | Heap + Index | Index-Organized |
|--------|--------------|-----------------|
| **Primary Key Lookup** | 2 I/Os | 1 I/O |
| **Range Scan on PK** | Random I/O | Sequential I/O |
| **Secondary Index** | Direct to row | Indirect (via PK) |
| **Insert Order** | Any order | PK order matters |
| **Table Size** | Fixed | Grows with index |

---

## Data Warehousing Storage

### Columnar Storage

Traditional row stores keep data together by row:
```
[Row 1: A,B,C][Row 2: D,E,F][Row 3: G,H,I]
```

Column stores keep data together by column:
```
Column A: [A,D,G]
Column B: [B,E,H]
Column C: [C,F,I]
```

### Why Columnar for Analytics?

**Analytical Query Pattern:**
```sql
SELECT AVG(salary), COUNT(*) 
FROM employees 
WHERE department = 'Engineering';
```

**Row Store:** Must read entire rows (all columns)
**Column Store:** Read only needed columns

### Compression in Column Stores

Columns have similar values, enabling excellent compression:

**Run-Length Encoding (RLE):**
```
Column values: [Engineering, Engineering, Engineering, Sales, Sales]
RLE: [(Engineering, 3), (Sales, 2)]
```

**Dictionary Compression:**
```
Unique values: {Engineering: 0, Sales: 1, Marketing: 2}
Compressed: [0, 0, 0, 1, 1]
```

**Bit-Packing:**
```
Values 0-255 need only 8 bits instead of 32
```

### Column Store Systems

| System | Characteristics |
|--------|-----------------|
| **Vertica** | Commercial, MPP |
| **Amazon Redshift** | Cloud, PostgreSQL-based |
| **Google BigQuery** | Serverless, Dremel-based |
| **ClickHouse** | Open source, fast |
| **Apache Parquet** | File format, not DBMS |

### Partitioning

Divide large tables into smaller, manageable pieces:

**Range Partitioning:**
```sql
CREATE TABLE sales (
    sale_id INT,
    sale_date DATE,
    amount DECIMAL
) PARTITION BY RANGE (sale_date) (
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01')
);
```

**Hash Partitioning:**
```sql
PARTITION BY HASH(customer_id) PARTITIONS 4;
```

**Benefits:**
- **Partition Pruning**: Skip irrelevant partitions
- **Parallel Processing**: Process partitions in parallel
- **Maintenance**: Archive or drop old partitions

---

## Summary

| Architecture | Best For | Examples |
|--------------|----------|----------|
| **Heap + Index** | General OLTP | PostgreSQL, SQL Server |
| **LSM-Tree** | Write-heavy workloads | RocksDB, Cassandra |
| **Index-Organized** | PK-heavy lookups | MySQL InnoDB |
| **Column Store** | Analytical queries | Redshift, ClickHouse |

---

## Key Takeaways

1. **LSM-Trees Excel at Writes**: Sequential writes, compaction handles cleanup
2. **Index-Organized Tables**: Fast PK access, secondary indexes indirect
3. **Column Stores for Analytics**: Read only needed columns, great compression
4. **Partitioning Improves Performance**: Pruning, parallelism, maintenance
5. **Choose Storage Based on Workload**: No one-size-fits-all solution

---

*Next Lecture: Storage Models & Compression - Row vs. Column Storage*
