# Lecture 06: Storage Models & Compression

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** September 15, 2025  
**Readings:** Database System Concepts, Chapter 11.2, 13.6  
**[📹 Watch Video](https://www.youtube.com/watch?v=yWnToWrskXE&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=6)**

---

## Storage Models Overview

The **storage model** determines how tuples are physically organized on disk. The two main approaches are **N-ary Storage Model (NSM)** and **Decomposition Storage Model (DSM)**.

---

## N-ary Storage Model (NSM / Row Store)

Traditional databases store data in **row-oriented** format, keeping all attributes of a tuple together.

### Example Table

| ID | Name  | Dept | Salary |
|----|-------|------|--------|
| 1  | Alice | CS   | 80K    |
| 2  | Bob   | EE   | 70K    |
| 3  | Carol | CS   | 90K    |

### Physical Layout

```
Page Layout:
+-------------------------------------------+
| [1|Alice|CS|80K] [2|Bob|EE|70K] [3|Carol|CS|90K] |
+-------------------------------------------+
  
  Record 1:  [ID:1][Name:Alice][Dept:CS][Salary:80K]
  Record 2:  [ID:2][Name:Bob][Dept:EE][Salary:70K]
  Record 3:  [ID:3][Name:Carol][Dept:CS][Salary:90K]
```

### Characteristics

| Aspect | Description |
|--------|-------------|
| **Tuple Access** | Single I/O to read entire row |
| **Column Access** | Must skip unwanted columns |
| **Insert/Update** | Fast (single location) |
| **Compression** | Moderate (diverse values) |
| **Cache Efficiency** | Good for row-oriented access |

### Best For

- **OLTP workloads** (Online Transaction Processing)
- Queries that access **most columns** of a row
- **Point lookups** and small range scans
- Frequent **inserts, updates, deletes**

### Example Query (Good for Row Store)

```sql
-- Accesses most columns
SELECT * FROM orders WHERE order_id = 12345;

-- Single-row insert
INSERT INTO orders VALUES (12346, '2024-01-15', 100, 'Alice', ...);
```

### Example Systems

PostgreSQL, MySQL (InnoDB), SQL Server, Oracle, SQLite

---

## Decomposition Storage Model (DSM / Column Store)

Column-oriented databases store each column separately.

### Physical Layout

```
ID Column:    [1|2|3]
Name Column:  [Alice|Bob|Carol]
Dept Column:  [CS|EE|CS]
Salary Column:[80K|70K|90K]
```

Each column is stored as a separate array or file.

### Characteristics

| Aspect | Description |
|--------|-------------|
| **Tuple Access** | Must read from multiple columns |
| **Column Access** | Single I/O for entire column |
| **Insert/Update** | Slower (multiple locations) |
| **Compression** | Excellent (similar values) |
| **Cache Efficiency** | Great for column-oriented access |

### Best For

- **OLAP workloads** (Online Analytical Processing)
- Queries that access **few columns** from many rows
- **Aggregation queries** (SUM, AVG, COUNT)
- **Data warehousing** and reporting

### Example Query (Good for Column Store)

```sql
-- Only accesses 2 columns
SELECT dept, AVG(salary) 
FROM employees 
GROUP BY dept;

-- Only reads Dept and Salary columns
```

### Example Systems

MonetDB, Vertica, Amazon Redshift, ClickHouse, Apache Parquet

---

## Comparison

| Aspect | Row Store | Column Store |
|--------|-----------|--------------|
| **Insert/Update** | Fast | Slow (multiple columns) |
| **Full Row Access** | Fast | Slow (reconstruct row) |
| **Column Access** | Slow (skip columns) | Fast |
| **Compression** | Moderate | Excellent |
| **Aggregation** | Moderate | Excellent |
| **Typical Use** | OLTP | OLAP |
| **Memory Bandwidth** | High (read unused data) | Low (read only needed) |

---

## Hybrid Approaches

### PAX (Partition Attributes Across)

Combines row and column storage within pages:

```
Page Structure:
+------------------+
| Mini-page 1: IDs |  ← All ID values for tuples in page
+------------------+
| Mini-page 2: Names |
+------------------+
| Mini-page 3: Depts |
+------------------+
| Mini-page 4: Salaries |
+------------------+
```

**Properties:**
- Within a page: data is column-oriented
- Across pages: data is row-oriented
- Balances OLTP and OLAP needs
- Used in some research systems

### HTAP (Hybrid Transactional/Analytical Processing)

Store data in **both** row and column formats:

```
Row Store:    For OLTP (current data)
Column Store: For OLAP (analytical queries)
              ↓
         Synchronization
```

**Systems:** SAP HANA, Oracle Database In-Memory, SQL Server Columnstore

**Trade-offs:**
- More storage required
- Synchronization overhead
- Best of both worlds for mixed workloads

---

## Compression Techniques

### Why Compression Matters

| Benefit | Explanation |
|---------|-------------|
| **Reduce I/O** | Less data to read from disk |
| **Increase Cache Efficiency** | More data fits in memory |
| **Save Storage** | Lower storage costs |
| **Improve Throughput** | Process more data per second |

### Dictionary Compression

Build a dictionary of unique values, store indices instead.

```
Original Column: [CS|EE|CS|CS|EE|ME|CS]

Step 1: Build Dictionary
  {CS: 0, EE: 1, ME: 2}

Step 2: Store Compressed Data
  [0|1|0|0|1|2|0]

Space Saved: 2 bytes × 7 values → 1 byte × 7 values + small dictionary
```

**Best For:** Low-cardinality columns (gender, status, department)

### Run-Length Encoding (RLE)

Store runs of identical values as (value, count) pairs.

```
Original: [A|A|A|A|B|B|C|C|C|C|C]

RLE: [(A,4)|(B,2)|(C,5)]

Space: 11 values → 3 pairs
```

**Best For:** Sorted columns or columns with natural runs

**Example in Column Stores:**
```
Sorted by date:
Date Column: [2024-01-01 × 1000][2024-01-02 × 1500][2024-01-03 × 800]
RLE: Very efficient!
```

### Bit-Packing

Use only the necessary bits for each value.

```
Values: 0-255 (need 8 bits, not 32)

Original: 32 bits × 1000 values = 4,000 bytes
Bit-packed: 8 bits × 1000 values = 1,000 bytes

Savings: 75%
```

**Best For:** Integer columns with small ranges

### Delta Encoding

Store differences between consecutive values.

```
Original: [1000|1005|1010|1012|1015]
Deltas:   [1000|+5|+5|+2|+3]

First value: 4 bytes
Deltas: 1 byte each (if small)
```

**Best For:** Sorted or time-series data

### Bit-Vector Encoding

For columns with very low cardinality, use bit-vectors.

```
Column: Gender [M|M|F|M|F|F|M]

Bit-vector M: [1|1|0|1|0|0|1]
Bit-vector F: [0|0|1|0|1|1|0]

Query: WHERE Gender = 'F'
→ Just read F bit-vector, find set bits
```

**Best For:** Boolean or low-cardinality columns

### Null Compression

Don't store actual null values.

```
Approach 1: Null Bitmap
  [bitmap: 10101] + [values for non-null]

Approach 2: Sentinels
  Use special value to indicate null (e.g., INT_MIN)
```

---

## Compression in Practice

### Column Store Compression Pipeline

```
Raw Column Data
      ↓
Dictionary Encoding (if low cardinality)
      ↓
Delta Encoding (if sorted)
      ↓
Bit-Packing (if small range)
      ↓
General Compression (LZ4, Zstd, Snappy)
      ↓
Compressed Storage
```

### Real-World Compression Ratios

| Workload | Compression Ratio |
|----------|-------------------|
| TPC-H (analytical) | 3-10x |
| Log data | 5-20x |
| Time-series | 10-50x |
| Text data | 2-5x |

---

## Summary

| Storage Model | Best For | Systems |
|---------------|----------|---------|
| **Row Store (NSM)** | OLTP, full-row access | PostgreSQL, MySQL |
| **Column Store (DSM)** | OLAP, aggregation | Redshift, ClickHouse |
| **Hybrid (PAX)** | Mixed workloads | Research systems |
| **HTAP** | Real-time analytics | SAP HANA, Oracle |

| Compression | Best For | Typical Savings |
|-------------|----------|-----------------|
| **Dictionary** | Low cardinality | 50-90% |
| **RLE** | Sorted/runs | 70-95% |
| **Bit-Packing** | Small integers | 50-75% |
| **Delta** | Time-series | 60-80% |

---

## Key Takeaways

1. **Row Stores for OLTP**: Fast inserts, full-row access
2. **Column Stores for OLAP**: Fast aggregations, great compression
3. **Hybrid Approaches**: Balance both workloads
4. **Compression is Critical**: Reduces I/O, improves cache efficiency
5. **Choose Compression Based on Data**: Different techniques for different patterns

---

*Next Lecture: Hash Tables - Fast Lookup Data Structures*
