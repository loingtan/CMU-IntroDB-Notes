# Lecture 25: Final Review + Systems Potpourri

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** December 3, 2025  
**Flash Talk:** Leonid Libkin (RelationalAI)  
**[📹 Watch Video](https://www.youtube.com/watch?v=qiVUf9X6ItM&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=25)**

---

## Course Review

This lecture provides a comprehensive review of all topics covered throughout the semester.

### Part I: Foundations & Data Models

| Lecture | Key Concepts |
|---------|-------------|
| **01: Relational Model** | Relations, keys, relational algebra (σ, π, ⋈) |
| **02: Modern SQL** | DDL, DML, joins, aggregation, window functions |

**Key Takeaway:** The relational model provides a mathematical foundation for databases. SQL is the practical implementation.

### Part II: Storage & Memory Management

| Lecture | Key Concepts |
|---------|-------------|
| **03: Storage I** | Pages, slotted pages, record layout |
| **04: Buffer Pool** | Page table, pin count, replacement policies |
| **05: Storage II** | LSM-trees, index-organized tables |
| **06: Compression** | Row vs column stores, compression techniques |

**Key Takeaway:** Disk-oriented databases use pages, buffer pools, and various storage models optimized for different workloads.

### Part III: Indexing & Data Structures

| Lecture | Key Concepts |
|---------|-------------|
| **07: Hash Tables** | Chaining, probing, dynamic hashing |
| **08-09: Indexes** | B+ trees, skip lists, Bloom filters |
| **10: Index Concurrency** | Latch coupling, B-link trees |

**Key Takeaway:** B+ trees are the dominant index structure. Hash tables excel at equality lookups.

### Part IV: Query Processing

| Lecture | Key Concepts |
|---------|-------------|
| **11: Sorting** | External merge sort, replacement selection |
| **12: Joins** | Nested loop, sort-merge, hash joins |
| **13-14: Execution** | Iterator model, vectorization, compilation |
| **15-16: Optimization** | Cost model, join ordering, DP |

**Key Takeaway:** Query optimization uses cost models and statistics to choose efficient execution plans.

### Part V: Transaction Processing

| Lecture | Key Concepts |
|---------|-------------|
| **17: CC Theory** | ACID, serializability, isolation levels |
| **18: 2PL** | Locks, strict 2PL, deadlock handling |
| **19: Timestamp Ordering** | Basic TO, OCC, snapshot isolation |
| **20: MVCC** | Version chains, garbage collection |

**Key Takeaway:** Concurrency control ensures correctness while allowing parallelism. MVCC enables non-blocking reads.

### Part VI: Recovery & Distributed Systems

| Lecture | Key Concepts |
|---------|-------------|
| **21: Logging** | WAL, log records, LSN |
| **22: Recovery** | ARIES: analysis, redo, undo |
| **23-24: Distributed** | Partitioning, 2PC, CAP theorem |

**Key Takeaway:** Logging and recovery ensure durability. Distributed systems trade consistency for availability.

---

## Emerging Topics in Database Systems

### NewSQL Databases

Combine SQL with horizontal scalability.

| System | Characteristics |
|--------|-----------------|
| **Google Spanner** | Global distribution, TrueTime |
| **CockroachDB** | PostgreSQL-compatible, distributed |
| **YugabyteDB** | Cassandra-like with SQL |

**Key Innovation:** Distributed transactions with strong consistency.

### Serverless Databases

Auto-scaling, pay-per-use.

| System | Characteristics |
|--------|-----------------|
| **Amazon Aurora Serverless** | MySQL/PostgreSQL compatible |
| **Google Cloud Spanner** | Global, strongly consistent |
| **PlanetScale** | MySQL, branch/schema migrations |

**Key Innovation:** Abstract infrastructure management.

### AI/ML in Databases

| Application | How It Works |
|-------------|--------------|
| **Learned Indexes** | Neural networks replace B-trees |
| **Query Optimization** | ML models predict cardinalities |
| **Automatic Tuning** | Learn optimal configurations |

**Example - Learned Index (RMI):**
```
Traditional: B-tree lookup O(log n)
Learned: Model predicts position O(1)

Model: position = f(key)
Refine with binary search if needed
```

### Specialized Hardware

| Hardware | Benefit |
|----------|---------|
| **GPU** | Parallel processing for analytics |
| **FPGA** | Custom query acceleration |
| **Persistent Memory** | Byte-addressable persistence |

**Systems:** OmniSci (GPU), Apache Arrow (columnar)

### Graph Databases

Optimized for relationship queries.

| System | Characteristics |
|--------|-----------------|
| **Neo4j** | Property graph model |
| **Amazon Neptune** | Fully managed |
| **TigerGraph** | Native parallel graph |

**Use Cases:** Social networks, fraud detection, recommendations

### Time-Series Databases

Optimized for timestamped data.

| System | Characteristics |
|--------|-----------------|
| **InfluxDB** | Purpose-built |
| **TimescaleDB** | PostgreSQL extension |
| **ClickHouse** | Fast analytical |

**Optimizations:** Time-based partitioning, compression, aggregation

---

## Key Principles Revisited

### Trade-offs are Everywhere

| Decision | Trade-off |
|----------|-----------|
| Row vs Column Store | OLTP vs OLAP |
| Locking vs MVCC | Overhead vs Concurrency |
| Sync vs Async Replication | Latency vs Durability |
| CP vs AP | Consistency vs Availability |

### Understand Your Workload

| Workload Type | Best Approach |
|---------------|---------------|
| **OLTP** | Row store, B+ trees, 2PL |
| **OLAP** | Column store, compression, vectorization |
| **HTAP** | Hybrid approaches |
| **Time-series** | Specialized storage, partitioning |

### Measure, Don't Guess

```
1. Use EXPLAIN to analyze plans
2. Profile actual execution
3. Benchmark with realistic data
4. Monitor in production
```

### Simplicity Wins

```
Complex optimizations must justify their cost:
  - Development time
  - Maintenance burden
  - Operational complexity
```

### Correctness First

```
Performance is secondary to data integrity:
  - ACID properties matter
  - Durability is non-negotiable
  - Test recovery procedures
```

---

## Summary

| Principle | Description |
|-----------|-------------|
| **Trade-offs** | Every decision has costs |
| **Workload matters** | Choose based on actual needs |
| **Measure** | Use data to guide decisions |
| **Simplicity** | Avoid unnecessary complexity |
| **Correctness** | Data integrity is paramount |

---

## Key Takeaways

1. **Comprehensive Foundation**: This course covers database internals end-to-end
2. **Theory Meets Practice**: Relational algebra → SQL → Implementation
3. **Systems Thinking**: Understand trade-offs at every level
4. **Stay Current**: Database technology continues to evolve
5. **Build Things**: Apply knowledge through projects

---

## Additional Resources

### Textbook
**Database System Concepts (7th Edition)**  
Abraham Silberschatz, Henry F. Korth, S. Sudarshan  
[https://www.db-book.com/](https://www.db-book.com/)

### Video Lectures
Full lecture playlist:  
[CMU 15-445 Fall 2025](https://www.youtube.com/playlist?list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5)

### BusTub DBMS
Course projects implemented in BusTub:  
[https://github.com/cmu-db/bustub](https://github.com/cmu-db/bustub)

---

*Course Complete! Thank you for learning with CMU 15-445.*
