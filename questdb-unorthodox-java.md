---
marp: true
theme: default
paginate: true
backgroundColor: #ffffff
color: #333333
style: |
  section {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }
  h1 {
    color: #0066cc;
  }
  h2 {
    color: #0052a3;
  }
  code {
    background-color: #f5f5f5;
    color: #333333;
  }
  pre {
    background-color: #f5f5f5;
    border: 1px solid #ddd;
  }
  blockquote {
    border-left: 4px solid #0066cc;
    padding-left: 1em;
    color: #555555;
  }
  .columns {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1em;
  }
---

# Unorthodox Java: Building QuestDB
## High-Performance Time-Series Database

**Jaromir Hamala**
QuestDB Core Engineering Team

![bg right:30% 80%](https://github.com/questdb/questdb/raw/main/.github/logo-readme.jpg)

---

# About Me

- **Jaromir Hamala** - QuestDB Core Engineering Team
- Passion for concurrency, performance, and distributed systems
- Working on QuestDB

---

# What is QuestDB?

- **Open-source time-series database** (Apache License 2.0)
- SQL with time-series extensions
- High-speed ingestion: InfluxDB line protocol (TCP/HTTP)
- Columnar storage format (native or Parquet)
- Partitioned and ordered by time

**GitHub:** https://github.com/questdb/questdb

---

# The Numbers 💪

## What do we mean by "high-performance"?

- **Millions** of rows ingested per second
- Query **billions** of rows efficiently
- Near-zero GC pauses

> No magic, just hard work and clever engineering

---

# Language Breakdown

<div class="columns">
<div>

## Implementation
- **90%** Java
- **10%** C++/Rust

</div>
<div>

## But... Unorthodox Java!
- In-house standard library
- Zero GC on hot path
- JNI when needed
- SIMD optimizations

</div>
</div>

---

# The Big Question

## Why Java at all?! 🤔

Let's explore why we chose Java and how we made it work for high-performance computing...

---

# QuestDB Design Principles

## Core Philosophy

1. **No Allocation on Hot-Path**
2. **No 3rd party Java libraries**
3. **Know your memory layout**
4. **Basic infrastructure MUST NOT allocate**

---

# What This Led To...

- **Own standard library** - Complete replacement of Java's stdlib
- **Strategic JNI usage** - JNI is NOT slow!
- **C-like code patterns** - Direct memory management
- **Object pooling** - Fast single-threaded pools for parsing

---

# Our Own Standard Library

![bg fit](stdlib.png)

---

# What Our Stdlib Provides

1. **I/O** - Network and file operations
2. **Collections** - Specialized for primitives (no boxing!)
3. **Strings** - CharSequence-based, not String
4. **Numbers** - Fast parsing/printing

**Zero dependencies on Java stdlib**

---

# Technique #1: Zero Allocation

## IPv4 to String Conversion
```sql
SELECT ip_address, CAST(ip_address AS STRING) as ip_str
```

```java
// Traditional: Creates garbage for each row
public String getIPv4String(Record rec) {
    return IPUtils.formatIPv4(arg.getIPv4(rec)); // New String per row!
}

// QuestDB: Reusable StringSink
private final StringSink sinkA = new StringSink();

public CharSequence getStrA(Record rec) {
    int ipv4 = arg.getIPv4(rec);  // Returns int
    sinkA.clear();  // Reset, don't allocate!
    Numbers.intToIPv4Sink(sinkA, ipv4);
    return sinkA;
}
```

---

# Technique #2: Off-Heap Memory

## Direct Memory Access
```java
// Allocate off-heap memory
long ptr = Unsafe.malloc(size);

// Direct memory operations
Unsafe.getUnsafe().putLong(ptr + offset, value);

// Manual memory management
Unsafe.free(ptr);
```

**Benefits:**
- No GC pressure
- Predictable memory layout
- Cache-friendly access patterns

---

# Technique #3: Custom JIT with SIMD

## Not Java Vector API - Our Own JIT!

**Built with:**
- **asmjit** library for code generation
- C++ backend, Java frontend
- AVX2 instructions for vectorization
- Processes 8 rows simultaneously (256-bit registers)

**Example:** Filter on INT column processes 8 values at once

---

# Technique #4: Our Own JIT Compiler! 🚀

## SQL JIT Architecture

**Frontend (Java):**
- Analyzes filter suitability
- Serializes AST to IR

**Backend (C++):**
- Uses **asmjit** library
- Emits x86-64 machine code
- AVX2 vectorization
- 58 instructions for simple filters!

---

# JIT Performance Impact

## Real Query Example
```sql
SELECT * FROM trips 
WHERE total_amount > 150 
AND pickup_datetime IN ('2009-01')
```

**Results on 13.5M rows:**
- Without JIT: 150ms (hot run)
- With JIT: 35ms (hot run)
- **76% reduction** in execution time
- **3.3 GB/s** filtering rate

---

# Pre-JIT vs JIT Filtering

<div class="columns">
<div>

## Pre-JIT (Java)
- Operator function call tree
- Row-by-row processing
- Virtual method calls
- Interpreted execution

</div>
<div>

## With JIT
- Direct machine code
- Vectorized (8 rows at once)
- No virtual calls
- Page frame processing

</div>
</div>

**11K lines of code, 250+ commits to build it!**

---

# Technique #5: Parallel GROUP BY Evolution

## The Journey to Scale

From single-threaded to massively parallel execution

---

# Single-threaded GROUP BY

```sql
SELECT sensor, max(temperature) FROM readings GROUP BY sensor
```

```
Input Data          HashMap
┌─────────┐        ┌──────────┐
│ Row 1   │───────►│ Key1: 10 │
│ Row 2   │        │ Key2: 25 │
│ Row 3   │        │ Key3: 15 │
│ ...     │        │ ...      │
│ Row N   │        └──────────┘
└─────────┘         
   ↓
Single Thread
```

**Problem:** Only uses one CPU core!

---

# Naive Parallel GROUP BY I

```
Input Data
┌─────────┐ worker1  ┌─────────┐
│Partition│─────────►│HashMap 1│
│   1     │          └─────────┘
├─────────┤ worker2  ┌─────────┐
│Partition│─────────►│HashMap 2│
│   2     │          └─────────┘
├─────────┤ worker3  ┌─────────┐
│Partition│─────────►│HashMap 3│
│   3     │          └─────────┘      
└─────────┘      
```


**Problem:** The same key is now in multiple maps!

---

# Naive Parallel GROUP BY II

```
Input Data
┌─────────┐ worker1  ┌─────────┐
│Partition│─────────►│HashMap 1│ \
│   1     │          └─────────┘  \
├─────────┤ worker2  ┌─────────┐   \   merge  ┌─────────┐
│Partition│─────────►│HashMap 2│    ──────────│  Result │
│   2     │          └─────────┘   /          └─────────┘
├─────────┤ worker3  ┌─────────┐  /     
│Partition│─────────►│HashMap 3│ /      
│   3     │          └─────────┘      
└─────────┘      
```


**Problem:** Merge becomes bottleneck with high cardinality!

---

# Sharded GROUP BY - The Solution

```
Thread 1          Thread 2          Thread 3
┌──────┐         ┌──────┐         ┌──────┐
│Shard0│         │Shard0│         │Shard0│──┐
│Shard1│         │Shard1│         │Shard1│  │ Parallel
│Shard2│         │Shard2│         │Shard2│  │ Merge!
│Shard3│         │Shard3│         │Shard3│──┘
└──────┘         └──────┘         └──────┘
   ↓                ↓                ↓
Key → Shard:    hash(key) & 3
```

Each key always goes to the same shard number across all threads!

---

# Sharded GROUP BY - How It Works

## Phase 1: Parallel Sharded Aggregation
```java
// Each worker thread
for (Row row : partition) {
    int shardId = hash(row.key) & (NUM_SHARDS - 1);
    shardMaps[shardId].aggregate(row);
}
```

## Phase 2: Parallel Shard Merging
```java
// Merge same shards from all workers - IN PARALLEL!
for (int shard = 0; shard < NUM_SHARDS; shard++) {
    parallelMerge(allWorkerShards[shard]);
}
```

**Result:** No single-threaded bottleneck!

---

# Memory Layout Matters

## Column Storage + Time Ordering
```
Traditional Row Storage:
[id|name|value|timestamp][id|name|value|timestamp]...

QuestDB Column Storage (sorted by time):
[id|id|id|id...] [name|name|name...] [value|value|value...]
↑                ↑                    ↑
Oldest → Newest  Oldest → Newest     Oldest → Newest
```

**Key Invariant:** Data is **physically sorted by time**

**Benefits:**
- Efficient time-based filtering (binary search!)
- Better cache utilization for recent data
- SIMD-friendly sequential access
- Natural data locality for time-series queries

---

# Object Pooling Strategy

## Single-threaded pools for parsing
```java
// SqlParser uses object pool for AST nodes
ObjectPool<ExpressionNode> expressionNodePool;

// Acquire nodes during parsing
ExpressionNode node = expressionNodePool.next();
node.configure(...);

// Mass release after plan creation
expressionNodePool.clear(); // O(1) - just reset position!
```

**Frontend optimization - parse without allocation**

---

# JNI is NOT Slow!

## The Secret: Pass Primitives Only!

```java
// SLOW: Passing object references
native void processData(String[] data);  // Object refs = slow!

// FAST: Passing memory addresses
native void processData(long address, int length);  // Just primitives!
```

## Off-heap data enables fast JNI
- Direct memory addresses (long)
- No object references
- No GC coordination needed
- Zero marshalling overhead

**This is why we use off-heap memory!**

---

# Real-World Performance

## Full Table Scan Example
**1.6 billion rows** - All taxi trips data

```sql
SELECT * FROM trips 
WHERE total_amount > 150 
AND passenger_count = 1
```

**With JIT:** Significant speedup even on cold runs!
**Peak filtering rate:** 9.4 GB/s (single thread)

---

# Key Optimizations

1. **Specialized Hash Tables**
   - Fixed-size keys (32/64-bit)
   - No generic overhead
   
2. **Vectorized SQL Functions**
   - LIKE operator with SIMD
   - Parallel filters
   
3. **Custom Memory Allocators**
   - Arena allocation
   - Zero fragmentation

---

# Time-Series Specifics

## SAMPLE BY Query
```sql
SELECT timestamp, avg(temperature)
FROM sensors
WHERE device_id = 'sensor1'
SAMPLE BY 1h
```

## Optimized for:
- Time-ordered access
- Recent data queries
- High-cardinality aggregations

---

# Lessons Learned

1. **Java CAN be fast** - With the right approach
2. **Measure everything** - Benchmarks guide optimization
3. **Question conventions** - Standard library isn't sacred
4. **Hardware sympathy** - Know your CPU and memory
5. **Batch operations** - Amortize costs

---

# Why Java After All?

## The Good
- **Excellent tooling** - Profilers, debuggers, IDEs
- **Strong type system** - Catches bugs early
- **JIT compiler** - Adaptive optimization
- **Mature ecosystem** - Libraries for non-critical paths
- **Developer productivity** - Fast iteration

## The Trade-offs
- Required deep JVM knowledge
- Built our own infrastructure
- Careful coding discipline

---

# Community & Resources

## Get Involved!
- **GitHub:** https://github.com/questdb/questdb
- **Community:** Active contributors welcome
- **Use cases:** IoT, Finance, Monitoring, Analytics

## Learn More
- QuestDB documentation
- Performance blog posts
- Benchmark results

---

# Q&A

## Thank you! 🙏

**Jaromir Hamala**
QuestDB Core Engineering Team

### Questions?

- Specific optimization techniques?
- Architecture decisions?
- Performance measurements?
- Getting started with QuestDB?

---

# Bonus: Code Examples

## Want to try QuestDB?

```bash
# Docker
docker run -p 9000:9000 questdb/questdb

# Enable JIT in server.conf
cairo.sql.jit.mode=on

# Java API
try (TableWriter writer = engine.getWriter("sensors")) {
    TableWriter.Row row = writer.newRow();
    row.putSym(0, "sensor1");
    row.putDouble(1, 23.5);
    row.append();
    writer.commit();
}
```

Visit: http://localhost:9000