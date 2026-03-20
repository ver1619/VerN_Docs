# ADR-002 WAL Buffer (Write Staging Layer)

### Status

Accepted

---

### Context

The WAL write path currently follows:

```java
WAL Writer → Layer A (disk append) → fsync
```

In this model:

- Each append may result in a syscall
- Small WriteBatches lead to inefficient I/O patterns
- Disk throughput is underutilized due to fragmented writes

Even with group commit:

- fsync cost is amortized
- But **write syscall overhead and fragmentation remain**

This creates a bottleneck:

> High-frequency small writes degrade overall throughput and increase CPU overhead

---

### Decision

Introduce a **WAL Buffer** as a staging layer between the WAL writer and physical log (Layer A).

---

### Definition

The WAL Buffer is an in-memory staging layer that:

- Temporarily accumulates WAL records
- Aggregates multiple writes before disk append
- Feeds batched data into Layer A

---

### Placement

```java
WAL Writer → WAL Buffer → Layer A → fsync
```

The WAL Buffer operates entirely within the write pipeline and is transparent to higher layers.

---

### Behavior

- WAL records are first written into the buffer
- Buffer accumulates data in append order
- Data is flushed from buffer to WAL segments based on:

  - Buffer capacity
  - Sync triggers (Layer C)
  - Commit group boundaries

---

### Constraints

- WAL Buffer must **not reorder records**
- WAL Buffer must **not alter durability semantics**
- WAL Buffer must not expose partial records to Layer A

---

### Durability Interaction

- Data in WAL Buffer is **not durable**
- Durability is achieved only after:

  - Data is appended to WAL (Layer A), AND
  - fsync completes

The buffer does not change the durability boundary.

---

### Consequences

#### Positive

- **Reduced syscall overhead**

  - Fewer write() calls to the OS

- **Improved I/O efficiency**

  - Larger sequential writes to disk

- **Better batching synergy**

  - Works naturally with group commit

- **Higher throughput under small writes**

  - Especially beneficial for KV workloads

---

#### Negative

- **Additional memory usage**

  - Buffer consumes in-memory space

- **Extra flush coordination**

  - Requires controlled draining of buffer

- **Potential latency trade-off**

  - Writes may wait in buffer before append

---

### Alternatives Considered

#### 1. Direct Write (No Buffer)

- WAL writer writes directly to disk

**Rejected because:**

- High syscall overhead
- Poor small-write performance

---

#### 2. OS Page Cache Only

- Rely entirely on OS buffering

**Rejected because:**

- Lack of control over batching
- Unpredictable flush behavior
- Weak alignment with WAL durability model

---

#### 3. Per-Write Immediate Flush

- Flush each write immediately

**Rejected because:**

- Negates batching benefits
- Extremely inefficient

---

### Rationale

The WAL Buffer introduces a controlled staging layer that:

```java
Reduces I/O overhead without affecting correctness
```

It complements:

- Group Commit → reduces fsync cost
- WAL Buffer → reduces write syscall cost

Together they optimize the full write path.

---

### Notes

- WAL Buffer is part of **Layer B execution flow**, not Layer A

- It is invisible to:

  - Recovery logic
  - Record structure
  - Lifecycle management

- It must preserve:

  - WAL ordering
  - Record atomicity

---

### Invariants Introduced

- WAL buffer preserves strict append order
- No record bypasses the buffer before WAL append
- Only fully formed records are flushed to Layer A
- Buffer contents are never considered durable

---

### Summary

WAL Buffer transforms:

```java
Many small writes → Fewer large sequential writes
```

Without changing:

```java
Durability guarantees, ordering, or recovery semantics
```
