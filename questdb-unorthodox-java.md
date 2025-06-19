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
QuestDB Engineering Team


---

# About Me

- **Jaromir Hamala** - QuestDB Engineering Team
- Passion for concurrency, performance, and distributed systems
- Working on QuestDB
- Before: Hazelcast, C2B2 - the birthplace of Payara AS

---

# What is QuestDB?

- **Open-source time-series database** (Apache License 2.0)
- SQL with time-series extensions
- PostgreSQL Wire Protocol compatible
- High-speed ingestion: InfluxDB line protocol
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
- **10%** C/C++/Rust

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

# Java is great! 

## It gives us

- **Community** - People who know it well
- **Tooling** - Excellent profilers, debuggers, IDEs
- **JIT Compiler** - Adaptive optimization at runtime 
- **Strong Type System** - Catches (some) bugs early
---

# But also: Rust wasn’t an option yet
```
commit 95b8095427c4e2c7814ad56d06b5fc65f6685130
Author: bluestreak01 <bluestreak@gmail.com>
Date:   Mon Apr 28 16:29:15 2014 -0700

    Initial commit
```

## **Rust 1.0**, was published on May 15, 2015

---
# QuestDB Design Principles

## Core Philosophy

1. No Allocation on Hot-Path
2. Know your memory layout
3. Core infrastructure MUST NOT allocate
4. No 3rd party Java libraries

## The founder has a background in High Frequency Trading

---
# Why Zero-GC?
## We have ZGC, Shenandoah, Azul C4 but...
- Concurrent collectors still cost CPU cycles
- And memory bandwidth
- And application throughput (barriers)

---

# What This Led To...

<div class="columns">
<div>

## Three Core Disciplines

1. **Memory Discipline** - Zero allocation, off-heap patterns
2. **Execution Discipline** - Custom JIT, runtime bytecode generation  
3. **Concurrency Discipline** - Lock-free algorithms, sharding
</div>
<div>


```
┌─────────────────────┐
│ Memory Discipline   │
├─────────────────────┤
│ • Zero Allocation   │
│ • Off-heap Memory   │
│ • Flyweight Pattern │
└─────────────────────┘
        ↓
┌─────────────────────┐
│Execution Discipline │
├─────────────────────┤
│ • Custom JIT        │
│ • Runtime Bytecode  │
│ • SIMD Operations   │
└─────────────────────┘
        ↓
┌──────────────────────┐
│Concurrency Discipline│
├──────────────────────┤
│ • Sharded GROUP BY   │
│ • Lock-free Design   │
│ • Single Writer      │
└──────────────────────┘
```
</div>
</div>


---

# Frontend vs Backend Memory Strategy

<div class="columns">
<div>

## Frontend
- **Parser, Planner, Optimizer**
- Pure Java objects (mostly)
- Uses **Object Pooling**
  - Example: AST nodes
- Fast allocation/deallocation

</div>
<div>

## Backend
- **Runtime, JIT, Storage**
- Uses **Off-heap Memory**
- Long-lived data
- Direct memory addresses
- Easy JNI communication

</div>
</div>

**Different parts, different strategies!**

---

![bg fit](stdlib.png)

---

# What Our Stdlib Provides

1. **I/O** - Network and file operations, including io_uring
2. **Collections** - Specialized for primitives (no boxing!)
3. **Strings** - CharSequence-based, not String
4. **Numbers** - Fast parsing/printing


---

# DISCIPLINE 1: MEMORY

## The Foundation of Performance

> "The fastest allocation is the one that never happens"

---
# Memory Technique 1: Zero Allocation

## Single-threaded pools
```java
// SqlParser uses object pool for AST nodes
ObjectPool<ExpressionNode> expressionNodePool;

// Acquire nodes during parsing
ExpressionNode node = expressionNodePool.next();
node.of(...);

// Mass release after plan creation
expressionNodePool.clear(); // O(1) - just reset position!
```

**Frontend optimization - parse without allocation**
**Demo: `SqlParser.parseTableName()`**

---


# Memory Technique 1b: Zero Allocation

## Postgres Wire Protocol and `double` columns
## **DEMO** - `PGConnectionContext.appendRecord()`

---
# Memory Technique 1c: Zero Allocation

IPv4 to String Conversion
```sql
SELECT ip_address, CAST(ip_address AS STRING) as ip_str
```

```java
// Traditional: Creates garbage for each row
public String getIPv4String(Record rec) {
    int ipv4 = arg.getIPv4(rec)
    String s = IPUtils.formatIPv4(ipv4); // New String per row!
    return s;
}

// QuestDB: Reusable StringSink
private final StringSink sinkA = new StringSink();
public CharSequence getStrA(Record rec) {
    sinkA.clear();  // Reset, don't allocate!
    Numbers.intToIPv4Sink(sinkA, arg.getIPv4(rec));
    return sinkA;
}
```

---

# Memory Technique 2: Off-Heap Memory

Direct Memory Access
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

# Memory-Mapped Files (mmap)

```java
// Map file directly into memory
long ptr = Files.mmap(fd, size, Files.MAP_RW);

// Read data directly from memory address
long value = Unsafe.getUnsafe().getLong(ptr + offset);

// Write data directly to memory address
Unsafe.getUnsafe().putLong(ptr + offset, newValue);

```
**Benefits:**
- **Zero-copy** - No data copying between kernel/userspace
- **Simple** - Kernel handles paging


---

# Flyweight Pattern with Off-heap

```java
public class FlyweightDirectUtf16Sink implements CharSequence {
    private long ptr;  // Start of memory region
    private long lo;   // Current position
    private long hi;   // End of memory region
    
    // Point to existing memory - no allocation!
    public FlyweightDirectUtf16Sink of(long start, long end) {
        this.ptr = start;
        this.lo = start;
        this.hi = end;
        return this;
    }
    
    // Direct memory access
    public char charAt(int index) {
        return Unsafe.getUnsafe().getChar(ptr + index * 2L);
    }
    
    [...]
}
```

**Key insight:** Same object, different memory regions!

---

## Memory Discipline Recap

- **No GC pressure** - Off-heap data is not tracked by the GC
- **Memory layout control** - Cache-friendly, predictable access
- **Zero allocation** - Reuse, don't recreate

---

# DISCIPLINE 2: EXECUTION

## Making Every CPU Cycle Count

> "Let the machine do what it does best"

---

# Memory Layout Matters

## Row vs Columnar Storage

---

# Traditional Row Storage

```
┌─────────────────────────────────────────────┐
│ Row 1: [id=1, sensor='A', temp=23.5, ts=t1] │
├─────────────────────────────────────────────┤
│ Row 2: [id=2, sensor='B', temp=24.1, ts=t2] │
├─────────────────────────────────────────────┤
│ Row 3: [id=3, sensor='A', temp=23.8, ts=t3] │
├─────────────────────────────────────────────┤
│ Row 4: [id=4, sensor='C', temp=22.9, ts=t4] │
└─────────────────────────────────────────────┘
```

**Problem for analytics:**
- To read all temperatures, must skip over other fields
- Poor cache utilization
- Can't use SIMD effectively

---

# Columnar Storage

```
id column:     [1, 2, 3, 4, ...]
sensor column: ['A', 'B', 'A', 'C', ...]
temp column:   [23.5, 24.1, 23.8, 22.9, ...]
ts column:     [t1, t2, t3, t4, ...]
```

**Benefits:**
- Read only what you need
- Sequential memory access
- Cache-friendly
- SIMD operations on entire columns

---

# QuestDB's Secret: Time Ordering

```
Traditional Columnar (unordered):
temp: [24.1, 22.9, 23.5, 23.8, ...]  ← Random time order

QuestDB Columnar (time-ordered):
temp: [22.9, 23.5, 23.8, 24.1, ...]
       ↑                        ↑
    Oldest                  Newest
```

**Key Invariant:** All columns are **physically sorted by time**

---

# Why Time Ordering Matters

## Efficient Time Filtering
```sql
SELECT avg(temp) FROM sensors 
WHERE ts > now() - '1h'
```

- **Binary search** to find time range start
- Sequential read of recent data
- No index needed!

## Cache Locality
- Recent data (most queried) stays hot in cache
- Natural prefetching for sequential access

---

# What is SIMD? Single Instruction, Multiple Data

**Traditional (Scalar):**
```
a[0] + b[0] = c[0]  ← One operation
a[1] + b[1] = c[1]  ← One operation  
a[2] + b[2] = c[2]  ← One operation
a[3] + b[3] = c[3]  ← One operation
```

**SIMD (Vectorized):**
```
┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
│a[0]│a[1]│a[2]│a[3]│ + │b[0]│b[1]│b[2]│b[3]│ = │c[0]│c[1]│c[2]│c[3]│
└───────────────────┘   └───────────────────┘   └───────────────────┘
               ↑ One instruction processes 4 values! ↑
```

**AVX2:** 256-bit registers = 8 ints or 4 doubles at once!
**AVX512:** 512-bit registers

---
# Explicit SIMD in Java I
```
JEP 338: Vector API (Incubator)
Authors	Vladimir Ivanov, Razvan Lupusoru, Paul Sandoz, Sandhya Viswanathan
Owner	Paul Sandoz
Type	Feature
Scope	JDK
Status	Closed / Delivered
Release	16
Component	hotspot / compiler
Discussion	panama dash dev at openjdk dot java dot net
Effort	M
Duration	M
Relates to	JEP 414: Vector API (Second Incubator)
Reviewed by	John Rose, Maurizio Cimadamore, Yang Zhang
Endorsed by	John Rose, Vladimir Kozlov
Created	2018/04/06 22:58
Updated	2021/08/28 00:15
Issue	8201271
```

---
# Explicit SIMD in Java II
```
JEP 508: Vector API (******Tenth****** Incubator)
Owner	Ian Graves
Type	Feature
Scope	JDK
Status	Closed / Delivered
Release	25
Component	core-libs
Discussion	panama dash dev at openjdk dot org
Effort	XS
Duration	XS
Relates to	JEP 489: Vector API (Ninth Incubator)
Reviewed by	Jatin Bhateja, Sandhya Viswanathan, Vladimir Ivanov
Endorsed by	Paul Sandoz
Created	2025/03/31 18:19
Updated	2025/05/21 21:28
Issue	8353296
```
---


# Execution Technique 1: Custom JIT with SIMD

## Not Java Vector API - Our Own JIT!

**Built with:**
- **asmjit** library for code generation
- C++ backend, Java frontend
- AVX2 instructions for vectorization
- Processes 8 rows simultaneously (256-bit registers)

**Example:** Filter on INT column processes 8 values at once

---

# Execution Technique 1b: JIT Architecture
<div class="columns">
<div>

## SQL JIT Architecture

**Frontend (Java):**
- Analyzes filter suitability
- Serializes AST to IR

**Backend (C++):**
- Uses **asmjit** library
- Emits x86-64 machine code
- AVX2 vectorization
</div>
<div>

```
SQL Query: WHERE total_amount > 150
                    ↓
┌─────────────────────────────────────────┐
│           Frontend (Java)               │
├─────────────────────────────────────────┤
│ 1. Parse filter expression              │
│ 2. Analyze suitability for JIT          │
│ 3. Build Abstract Syntax Tree (AST)     │
│ 4. Serialize to Intermediate Rep (IR)   │
└─────────────────┬───────────────────────┘
                  │ IR + Metadata
                  ↓
┌─────────────────────────────────────────┐
│           Backend (C++)                 │
├─────────────────────────────────────────┤
│ 1. Parse IR from Java frontend          │
│ 2. Generate x86-64 machine code         │
│ 3. Use AVX2 SIMD instructions           │
│ 4. Return function pointer to Java      │
└─────────────────┬───────────────────────┘
                  │ Native function pointer
                  ↓
    Vectorized filter processes 8 rows at once!
```
</div>
</div>

---

![bg fit](ir.png)

---

![bg fit](jit.png)

---

# JIT Performance Impact

## Real Query Example
```sql
SELECT * FROM trips 
WHERE total_amount > 150 
AND pickup_datetime IN ('2009-01')
```

**Single-thread scanning 13.5M rows (out of 1.6B total rows):**
- Without JIT: 150ms (hot run)
- With JIT: 35ms (hot run)
- **76% reduction** in execution time
- **3.3 GB/s** filtering rate

## Live DEMO
---

# Pre-JIT Filtering

- Operator function call tree
- Row-by-row processing
- Virtual method calls
- Interpreted execution

```java
public boolean hasNext() {
    while (base.hasNext()) {
        if (filter.getBool(record)) {
            return true;
        }
    }
    return false;
}
```

![bg fit right ](tree.webp)

---

# JIT Filtering

- Direct machine code
- Vectorized (8 rows at once)
- No virtual calls


![bg fit right ](filter.png)

**11K lines of code, 250+ commits to build it!**

---


# Execution Technique 2: Runtime Bytecode Generation

## Custom Comparators for ORDER BY

```sql
SELECT * FROM readings 
ORDER BY sensor_id, batch_id DESC, timestamp
```

**Problem:** Generic comparator with virtual calls is slow!

**Solution:** Generate specialized bytecode at runtime

---

# Traditional Generic Comparator

```java
public int compare(Record a, Record b, int[] columns, int[] types) {
    for (int i = 0; i < columns.length; i++) {
        int col = Math.abs(columns[i]) - 1;
        boolean desc = columns[i] < 0;
        int cmp = 0;
        
        switch (types[col]) {
            case STRING:
                cmp = compareString(a.getStr(col), b.getStr(col));
                break;
            case LONG:
                cmp = Long.compare(a.getLong(col), b.getLong(col));
                break;
            case DOUBLE:
                cmp = Double.compare(a.getDouble(col), b.getDouble(col));
                break;
            // ... 20+ more types!
        }
        if (desc) cmp = -cmp;
        if (cmp != 0) return cmp;
    }
    return 0;
}
```

**Problems:** Type checking overhead, virtual calls, no optimization

---

# QuestDB: Runtime Bytecode Generation

![bg](comparator.png)

---

# Generated Class Structure

```java
// Generated class for ORDER BY sensor_id, batch_id DESC, timestamp
public class GeneratedComparator implements RecordComparator {
    
    // Fields to cache left record values
    private int f0;    // sensor_id column
    private int f1;    // batch_id column  
    private long f2;   // timestamp column
    
    // Cache left record values
    public void setLeft(Record record) {
        this.f0 = record.getInt(0);
        this.f1 = record.getInt(1);
        this.f2 = record.getLong(2);
    }
    
    // Compare cached left with right record
    public int compare(Record right) {
        int cmp = Integer.compare(this.f0, right.getInt(0));
        if (cmp != 0) return cmp;
        cmp = -Integer.compare(this.f1, right.getInt(1)); // DESC!
        if (cmp != 0) return cmp;
        return Long.compare(this.f2, right.getLong(2));
    }
}
```

---
# QuestDB: Runtime Bytecode Generation


```java
// Generated at runtime for ORDER BY sensor_id, batch_id DESC, timestamp
public int compare(Record r) {
    int cmp = Integer.compare(this.f0, r.getInt(0));
    if (cmp != 0) return cmp;
    cmp = -Integer.compare(this.f1, r.getInt(1));
    if (cmp != 0) return cmp;
    return Long.compare(this.f2, r.getLong(2));
}
```

**RecordComparatorCompiler generates:**
- Custom class per query
- Type-specific comparison inlined
- No switches, no virtual calls, no boxing
- JVM can optimize this perfectly!

---

---

# DISCIPLINE 3: CONCURRENCY

## Scaling Without Contention

> "Share nothing, merge something"

---

# Parallel GROUP BY Evolution

## The Journey to Scale

From single-threaded to massively parallel execution

---

# Single-threaded GROUP BY

```sql
SELECT sensor, max(temperature) FROM readings GROUP BY sensor
```

```
Input Data                  Output Map
┌─────────┐                ┌──────────┐
│ NYC, 23 │───────────────►│ NYC: 23  │
│ SFO, 32 │  Single Worker │ SFO: 32  │
│ NYC, 21 │                │ ...      │
│ NYC, 22 │                │ ...      │
│ ...     │                │ ...      │
│ Row N   │                └──────────┘
└─────────┘         
```

**Problem:** Only uses one CPU core!

---

# Concurrent GROUP BY?

```sql
SELECT sensor, max(temperature) FROM readings GROUP BY sensor
```

```
Input Data
┌─────────┐ worker0  ┌────────────────┐
│Partition│─────────►│ SFO: 32        │
│   1     │          │ NYC: 23        │
├─────────┤ worker1  │ ....           │
│Partition│─────────►│ ....           │ 
│   2     │          │ ....           │
├─────────┤ worker2  │ ....           │ 
│Partition│─────────►│ ....           │
│   3     │          └────────────────┘      
└─────────┘            Concurrent Map
```

**Problem:** Contention on shared map
## Single Writer Principle Violated!

---


# Naive Parallel GROUP BY I

```
Input Data
┌─────────┐ worker0  ┌─────────┐
│Partition│─────────►│HashMap 1│
│   1     │          └─────────┘
├─────────┤ worker1  ┌─────────┐
│Partition│─────────►│HashMap 2│
│   2     │          └─────────┘
├─────────┤ worker2  ┌─────────┐
│Partition│─────────►│HashMap 3│
│   3     │          └─────────┘      
└─────────┘      
```


**Problem:** The same key is now in multiple maps!

---

# Naive Parallel GROUP BY II

```
Input Data
┌─────────┐ worker0  ┌─────────┐
│Partition│─────────►│HashMap 1│ \
│   1     │          └─────────┘  \
├─────────┤ worker1  ┌─────────┐   \   merge  ┌─────────┐
│Partition│─────────►│HashMap 2│    ──────────│  Result │
│   2     │          └─────────┘   /          └─────────┘
├─────────┤ worker2  ┌─────────┐  /     
│Partition│─────────►│HashMap 3│ /      
│   3     │          └─────────┘      
└─────────┘      
```


**Problem:** Merge becomes bottleneck with high cardinality!

---

# Solution: Sharded GROUP BY

```
Each worker creates multiple small maps (shards):

  Worker 0          Worker 1         Worker 2         Worker 3
┌──────────┐      ┌──────────┐     ┌──────────┐     ┌──────────┐
│Map Shard0│      │Map Shard0│     │Map Shard0│     │Map Shard0│
│Map Shard1│      │Map Shard1│     │Map Shard1│     │Map Shard1│
│Map Shard2│      │Map Shard2│     │Map Shard2│     │Map Shard2│
│Map Shard3│      │Map Shard3│     │Map Shard3│     │Map Shard3│
└──────────┘      └──────────┘     └──────────┘     └──────────┘

Key → Shard: hash(key) % 4
```

**Key property:** Each key always maps to the same shard number!

---

# Sharded GROUP BY - Parallel Merge

```
 Thread 0   Thread 1    Thread 2    Thread 3       Final Result
┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐       ┌─────────┐
│Shard0│────│Shard0│────│Shard0│────│Shard0│──────►│ Result0 │
└──────┘    └──────┘    └──────┘    └──────┘       └─────────┘
┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐       ┌─────────┐
│Shard1│────│Shard1│────│Shard1│────│Shard1│──────►│ Result1 │
└──────┘    └──────┘    └──────┘    └──────┘       └─────────┘
┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐       ┌─────────┐
│Shard2│────│Shard2│────│Shard2│────│Shard2│──────►│ Result2 │
└──────┘    └──────┘    └──────┘    └──────┘       └─────────┘
┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐       ┌─────────┐
│Shard3│────│Shard3│────│Shard3│────│Shard3│──────►│ Result3 │
└──────┘    └──────┘    └──────┘    └──────┘       └─────────┘
```

**4 parallel merges instead of 1!** No key appears in multiple results.
**Result:** No single-threaded bottleneck!

---

# The Three Disciplines - Recap

<div class="columns">
<div>

## Memory Discipline ✓
- Zero allocation patterns
- Off-heap memory management
- Flyweight views

## Execution Discipline ✓
- Custom JIT compiler
- Runtime bytecode generation
- SIMD vectorization

</div>
<div>

## Concurrency Discipline ✓
- Sharded algorithms
- Lock-free design
- Single writer principle

## Result
**Millions of rows/sec ingestion**
**Billions of rows queries**
**Near-zero GC pauses**

</div>
</div>

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
2. **Hardware sympathy** - Know your CPU and memory
3. **Question conventions** - Standard library isn't sacred
4. **Batch operations** - Amortize costs
5. **Measure everything** - Benchmarks guide optimization

---

# Why Java After All?

## The Good
- **Excellent tooling** - Profilers, debuggers, IDEs
- **JIT compiler** - Adaptive optimization
- **Developer productivity** - Fast iteration

## The Trade-offs
- Required deep JVM knowledge
- Built our own infrastructure
- Careful coding discipline - **super important!**

---

# Is it worth it?

- **Maybe?** It works for us!
- **Depends on your use case** - High-performance, low-latency systems
- **Java is not the bottleneck** - It's how you use it!
- **Unorthodox Java** - Yes, but it can be done!

---

# Community & Resources

## Get Involved!
- **GitHub:** https://github.com/questdb/questdb
- **Community:** Active contributors welcome
- **Use cases:** Finance, IoT, Monitoring, Analytics

## Learn More
- QuestDB documentation
- Performance blog posts
- Benchmark results

---

# Q&A

## Thank you! 🙏

**Jaromir Hamala**
QuestDB Engineering Team

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
```

Visit: http://localhost:9000