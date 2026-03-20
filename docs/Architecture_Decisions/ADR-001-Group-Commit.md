# ADR-001 Group Commit

### Status

Accepted

---

### Context

The WAL durability model requires that:
- Every acknowledged write must satisfy the **durability boundary** (Layer C)
- WAL append operations are strictly ordered and serialized (Layer B)
- Durability is enforced via **fsync**, which is an expensive operation

Under a naïve approach (sync per write):
- Each WriteBatch triggers an independent fsync
- System throughput becomes bounded by fsync latency
- High write concurrency leads to severe performance degradation

At the same time, relaxing durability (fully asynchronous writes) violates:

- The WAL guarantee: no acknowledged write is lost

Therefore, the system must:

> Improve throughput while preserving strict durability guarantees

---

### Decision

Introduce **Group Commit** as part of the WAL durability policy (Layer C).

---

### Definition

A **commit group** is set of Writebatches that:

- Are appended to WAL in order
- Share a single durability operation (fsync)
- Are acknowledged together only after durability is achieved

---

### Formation

Commit groups are formed by the WAL writer based on:

- Write queue availability
- WAL buffer state
- Sync policy triggers (time / size / hybrid)

---

### Boundary Rule

A commit group is closed when:
- A durability operation is triggered, OR
- No additional writes are available for batching

---

### Durability Semantics

- All writes in a commit group become **durable simultaneously**
- ACK is issued **only after fsync completes**
- No write may be acknowledged independently of its commit group

---

### Ordering Constraint

- Commit group ordering must strictly follow WAL append order
- No reordering across groups is allowed
- Sequence number monotonicity must be preserved

---

### Consequences

#### Positive

- **Throughput improvement**
  - Multiple writes amortize a single fsync cost
- **Better disk utilization**
  - Larger sequential writes improve I/O efficiency
- **Scalability under concurrency**
  - High write concurrency no longer directly maps to fsync frequency

---

#### Negative

- **Increased latency variability**
  - Writes may wait for group formation before durability
- **Batching-induced delay**
  - Low-throughput workloads may experience slight delays
- **More complex commit coordination**
  - WAL writer must manage grouping boundaries

---

### Alternatives Considered

#### 1. Sync Per Write

- fsync after every WriteBatch

**Rejected because:**

- Extremely poor throughput
- Not viable under concurrent workloads

---

#### 2. Fully Asynchronous Writes

- Acknowledge writes before durability

**Rejected because:**

- Violates WAL durability guarantees
- Acknowledged writes may be lost

---

#### 3. Fixed-Size Batching Only

Batch based only on size threshold

**Rejected because:**

- Inflexible under varying workloads
- Does not adapt to time-sensitive latency requirements

---

### Rationale

Group commit provides the optimal balance:
```java
Durability Guarantees + Throughput Efficiency
```
It preserves:
- WAL correctness model
- Deterministic recovery
- ACK contract

While significantly improving:
- Write throughput
- Resource utilization

---

### Notes

- Group commit is integrated into **Layer C (Durability Policy)**
- It is enabled by:
  - WAL Buffer (staging layer)
  - Single WAL writer (serialization point)
- The mechanism does not alter:
  - WAL ordering
  - Record structure
  - Recovery semantics

---

### Invariants Introduced

- No write is acknowledged before its commit group is durable
- Commit group ordering equals WAL append order
- All writes in a commit group share identical durability state

---

### Summary
Group Commit transforms:
```java
Many writes → One durability event
```
Without weakening:
```java
Durability guarantees or recovery correctness
```