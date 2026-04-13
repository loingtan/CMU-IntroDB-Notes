# Lecture 13: Query Execution I

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** October 20, 2025  
**Readings:** Database System Concepts, Chapter 15.1-15.3, 15.7  
**Project:** Query Execution

---

## Query Execution Architecture

### Processing Pipeline

```
SQL Query
    ↓
Parser → Parse Tree
    ↓
Binder → Logical Plan (with types, table refs)
    ↓
Rewriter → Transformed Logical Plan
    ↓
Optimizer → Physical Plan
    ↓
Executor → Results
```

---

## Iterator Model (Volcano Model)

The most common query execution model.

### Operator Interface

```cpp
class Operator {
public:
    virtual void Init() = 0;
    virtual Tuple* Next() = 0;  // Returns tuple or nullptr
    virtual void Close() = 0;
};
```

### Example: Simple Query

```sql
SELECT * FROM R WHERE a > 5
```

```cpp
// Relational Algebra: σ_{a>5}(R)

scan = new TableScan("R");
select = new Selection(scan, [](Tuple* t) { return t->Get("a") > 5; });

Tuple* tuple;
while ((tuple = select->Next()) != nullptr) {
    Output(tuple);
}
```

### Operator Tree

```
        Selection (a > 5)
              |
        Table Scan (R)
              |
            Disk
```

**Execution Flow:**
1. `Selection::Next()` calls `TableScan::Next()`
2. `TableScan::Next()` reads from disk
3. `Selection::Next()` applies predicate, returns if match
4. Repeat until no more tuples

### Example: Join Query

```sql
SELECT * FROM R, S WHERE R.a = S.b
```

```cpp
// Relational Algebra: R ⋈_{a=b} S

scan_r = new TableScan("R");
scan_s = new TableScan("S");
join = new NestedLoopJoin(scan_r, scan_s, "a", "b");

Tuple* tuple;
while ((tuple = join->Next()) != nullptr) {
    Output(tuple);
}
```

### Operator Tree

```
        Nested Loop Join
           /        \
    Table Scan    Table Scan
        (R)          (S)
```

---

## Operator Implementations

### Scan Operators

**Sequential Scan:**
```cpp
class SeqScan : public Operator {
    Table* table;
    Iterator it;
    
    void Init() { it = table->begin(); }
    
    Tuple* Next() {
        if (it == table->end()) return nullptr;
        return *(it++);
    }
};
```

**Index Scan:**
```cpp
class IndexScan : public Operator {
    Index* index;
    Key low_key, high_key;
    Iterator it;
    
    void Init() {
        it = index->RangeScan(low_key, high_key);
    }
    
    Tuple* Next() {
        if (!it.HasNext()) return nullptr;
        RID rid = it.Next();
        return table->Get(rid);
    }
};
```

### Selection Operator

```cpp
class Selection : public Operator {
    Operator* child;
    Predicate* pred;
    
    void Init() { child->Init(); }
    
    Tuple* Next() {
        Tuple* tuple;
        while ((tuple = child->Next()) != nullptr) {
            if (pred->Evaluate(tuple)) {
                return tuple;
            }
        }
        return nullptr;
    }
};
```

### Projection Operator

```cpp
class Projection : public Operator {
    Operator* child;
    vector<string> output_cols;
    
    Tuple* Next() {
        Tuple* tuple = child->Next();
        if (tuple == nullptr) return nullptr;
        
        Tuple* result = new Tuple();
        for (auto& col : output_cols) {
            result->Set(col, tuple->Get(col));
        }
        return result;
    }
};
```

### Join Operators

**Nested Loop Join:**
```cpp
class NestedLoopJoin : public Operator {
    Operator* left;
    Operator* right;
    string left_key, right_key;
    Tuple* left_tuple;
    
    void Init() {
        left->Init();
        right->Init();
        left_tuple = left->Next();
    }
    
    Tuple* Next() {
        while (left_tuple != nullptr) {
            Tuple* right_tuple;
            while ((right_tuple = right->Next()) != nullptr) {
                if (left_tuple->Get(left_key) == right_tuple->Get(right_key)) {
                    return Combine(left_tuple, right_tuple);
                }
            }
            // Exhausted right, get next left
            right->Init();  // Reset right scan
            left_tuple = left->Next();
        }
        return nullptr;
    }
};
```

---

## Materialization Model

Alternative to iterator model: each operator processes all input, produces all output.

```cpp
class Operator {
    virtual vector<Tuple*> Execute() = 0;
};
```

### Example

```cpp
class Selection : public Operator {
    vector<Tuple*> Execute() {
        vector<Tuple*> input = child->Execute();
        vector<Tuple*> output;
        for (auto& tuple : input) {
            if (pred->Evaluate(tuple)) {
                output.push_back(tuple);
            }
        }
        return output;
    }
};
```

### Comparison

| Aspect | Iterator Model | Materialization |
|--------|----------------|-----------------|
| **Memory** | Low (one tuple) | High (all tuples) |
| **Pipeline** | Natural | Blocked |
| **Overhead** | Function calls per tuple | Batch processing |
| **Best for** | OLTP, streaming | OLAP with few results |

---

## Vectorization Model

Process batches of tuples (vectors) for better performance.

```cpp
class Operator {
    virtual Batch* Next() = 0;  // Returns batch of tuples
};

struct Batch {
    static const int BATCH_SIZE = 1024;
    Tuple* tuples[BATCH_SIZE];
    int count;
};
```

### Example: Vectorized Selection

```cpp
class VectorizedSelection : public Operator {
    Batch* Next() {
        Batch* output = new Batch();
        output->count = 0;
        
        Batch* input = child->Next();
        if (input == nullptr) return nullptr;
        
        for (int i = 0; i < input->count; i++) {
            if (pred->Evaluate(input->tuples[i])) {
                output->tuples[output->count++] = input->tuples[i];
            }
        }
        
        return output;
    }
};
```

### Benefits

| Benefit | Explanation |
|---------|-------------|
| **Cache efficiency** | Process multiple tuples together |
| **SIMD** | Vector instructions on batches |
| **Reduced overhead** | Fewer function calls |
| **Better compression** | Columnar batch processing |

### Systems Using Vectorization

VectorWise, MonetDB, DuckDB, modern column stores

---

## Summary

| Model | Unit of Processing | Best For |
|-------|-------------------|----------|
| **Iterator** | One tuple | OLTP, general purpose |
| **Materialization** | All tuples | OLAP, small results |
| **Vectorization** | Batch of tuples | OLAP, column stores |

---

## Key Takeaways

1. **Iterator Model**: Clean, composable, widely used
2. **Pull-Based**: Parent pulls tuples from children
3. **Pipeline**: Tuples flow through operators
4. **Vectorization**: Modern approach for analytics
5. **Choose Model Based on Workload**: OLTP vs OLAP

---

*Next Lecture: Query Execution II - Advanced Topics*
