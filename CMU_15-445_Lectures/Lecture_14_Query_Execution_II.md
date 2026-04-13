# Lecture 14: Query Execution II

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** October 22, 2025  
**Readings:** Database System Concepts, Chapter 22  
**Homework:** Execution & Planning  
**Flash Talk:** Will Manning (SpiralDB)

---

## Parallel Query Execution

### Types of Parallelism

| Type | Description |
|------|-------------|
| **Intra-operator** | Divide single operator among threads |
| **Inter-operator** | Pipeline multiple operators |
| **Bushy** | Execute independent subtrees in parallel |

### Intra-Operator Parallelism

```
Parallel Scan:
+--------+  +--------+  +--------+  +--------+
| Scan 1 |  | Scan 2 |  | Scan 3 |  | Scan 4 |
| P0-P24 |  | P25-49 |  | P50-74 |  | P75-99 |
+--------+  +--------+  +--------+  +--------+
     |           |           |           |
     +-----------+-----------+-----------+
                 |
            Exchange Operator
                 |
            Merge Results
```

**Exchange Operators:**
- **Gather**: Collect results from multiple workers
- **Distribute**: Partition data to workers
- **Repartition**: Redistribute based on different key

### Inter-Operator Parallelism

```
Pipeline:

Scan → Filter → Project → Output
  |       |         |
Thread  Thread   Thread

Each operator runs in its own thread
Tuples flow through pipeline
```

### Bushy Parallelism

```
        Join
       /    \
    Join    Join
    /  \    /  \
   A    B  C    D

Join A-B and Join C-D can execute in parallel
Then final join combines results
```

---

## Query Compilation

Modern systems compile queries to machine code for better performance.

### Why Compilation?

| Benefit | Explanation |
|---------|-------------|
| **Eliminate interpretation** | No virtual function calls |
| **Better inlining** | Combine operator logic |
| **Loop unrolling** | Process multiple tuples |
| **Branch prediction** | Better CPU utilization |
| **SIMD** | Vector instructions |

### Compilation Approaches

#### 1. LLVM IR Generation

```cpp
// Generate LLVM intermediate representation
Function* GenerateQueryFunction() {
    Function* func = CreateFunction();
    
    // Generate code for scan
    Value* tuple = GenerateScan();
    
    // Generate code for filter
    Value* cond = GenerateFilter(tuple);
    
    // Generate conditional output
    GenerateIf(cond, [&]() {
        GenerateOutput(tuple);
    });
    
    return func;
}
```

**Systems:** HyPer, Umbra, DuckDB

#### 2. C++ Code Generation

```cpp
// Generate C++ source code
string GenerateQuery() {
    return R"(
        void Execute() {
            for (auto& tuple : table) {
                if (tuple.a > 5) {
                    output(tuple);
                }
            }
        }
    )";
}

// Compile to shared library
// Load and execute
```

**Systems:** MemSQL, some research systems

#### 3. Direct Machine Code

Emit assembly instructions directly.

```cpp
void EmitAdd(Register dst, Register src1, Register src2) {
    // Emit x86 ADD instruction
    code[pc++] = 0x01;  // ADD opcode
    code[pc++] = ModRM(dst, src1, src2);
}
```

### Compilation Example

```sql
SELECT a, b FROM R WHERE a > 5
```

**Interpreted:**
```cpp
while (tuple = scan->Next()) {
    if (filter->Evaluate(tuple)) {
        output(project->Evaluate(tuple));
    }
}
```

**Compiled:**
```cpp
void Execute() {
    for (int i = 0; i < num_tuples; i++) {
        int a = data_a[i];
        int b = data_b[i];
        if (a > 5) {
            output_buffer[count++] = {a, b};
        }
    }
}
```

### Performance Impact

| Workload | Speedup |
|----------|---------|
| Simple queries | 2-5x |
| Complex queries | 5-20x |
| Analytical queries | 10-100x |

---

## Adaptive Query Execution

### Problem

Statistics may be inaccurate, leading to bad plans.

```
Optimizer estimates: 100 rows
Actual: 1,000,000 rows

Bad plan chosen → Slow execution
```

### Solutions

#### Runtime Statistics

```
During execution:
  - Collect actual cardinalities
  - Compare with estimates
  - Re-optimize if significantly different
```

#### Plan Re-optimization

```
Execute partial plan
  ↓
Collect statistics
  ↓
Re-optimize remaining plan
  ↓
Continue with new plan
```

#### Robust Query Optimization

```
Instead of optimizing for expected cost:
  - Optimize for worst-case cost
  - Choose plan that performs well across scenarios
```

---

## Query Execution Best Practices

### For Database Developers

1. **Use appropriate indexes**
2. **Avoid SELECT ***
3. **Filter early**
4. **Limit result sets**
5. **Use EXPLAIN to analyze plans**

### For Query Optimizers

1. **Accurate statistics**
2. **Good cost model**
3. **Consider multiple plans**
4. **Adaptive execution**
5. **Compiled execution for complex queries**

---

## Summary

| Topic | Key Idea |
|-------|----------|
| **Intra-operator Parallelism** | Divide operator work |
| **Inter-operator Parallelism** | Pipeline operators |
| **Query Compilation** | Generate machine code |
| **Adaptive Execution** | Adjust based on runtime stats |

---

## Key Takeaways

1. **Parallelism Improves Throughput**: Multiple strategies available
2. **Compilation Eliminates Overhead**: Significant speedup possible
3. **Adaptivity Handles Uncertainty**: Runtime adjustments
4. **No Silver Bullet**: Combine techniques appropriately

---

*Next Lecture: Query Planning & Optimization I*
