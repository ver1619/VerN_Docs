# ADR-003 Adaptive Sync (Workload-Aware Durability Policy)

### Status

Accepted

---

### Context

The WAL durability model enforces:

- Strict durability boundary (Layer C)
- ACK only after durability condition is satisfied
- fsync as the mechanism to ensure persistence

However, fsync introduces a fundamental trade-off:

| Mode           | Behavior        | Problem                      |
| -------------- | --------------- | ---------------------------- |
| Sync per batch | Low latency     | Poor throughput under load   |
| Delayed sync   | High throughput | Increased durability latency |

A fixed sync policy leads to inefficiency:

- Under **low load** → unnecessary batching delays
- Under **high load** → excessive fsync frequency limits throughput

Therefore, the system requires:

> A dynamic mechanism to balance latency and throughput without violating durability guarantees

---

### Decision

Introduce **Adaptive Sync** as an extension of Layer C (Durability Policy).

---

### Definition

Adaptive Sync allows the system to dynamically adjust **sync timing behavior** based on runtime workload conditions.

It does **not** change:

- Durability definition
- ACK contract
- Recovery semantics

It only adjusts:

> *When durability is triggered*

---

### Behavior

The system may adapt sync frequency based on:

- Write arrival rate
- Queue pressure
- WAL buffer utilization
- Commit group formation

---

### Operational Modes (Conceptual)

#### Low Load

- Smaller commit groups
- More frequent sync
- Lower write latency

---

#### High Load

- Larger commit groups
- Less frequent sync
- Higher throughput

---

### Constraints

- Adaptive behavior must not violate durability boundary
- ACK must still occur only after durability is achieved
- No write may be acknowledged prematurely

---

### Interaction with Group Commit

- Adaptive Sync directly influences **commit group size and timing**
- Larger groups under high load → better fsync amortization
- Smaller groups under low load → lower latency

---

### Interaction with WAL Buffer

- WAL Buffer provides staging capacity for adaptive batching
- Sync decisions may consider buffer state

---

### Consequences

#### Positive

- **Improved throughput under high load**

  - Reduces fsync frequency

- **Reduced latency under low load**

  - Avoids unnecessary batching delay

- **Better resource utilization**

  - Aligns I/O behavior with workload

---

#### Negative

- **Non-uniform latency characteristics**

  - Write latency becomes workload-dependent

- **Increased policy complexity**

  - Sync behavior is no longer static

- **Harder predictability**

  - Exact sync timing varies dynamically

---

### Alternatives Considered

#### 1. Fixed Sync Policy

- Static time/size thresholds

**Rejected because:**

- Inefficient across varying workloads
- Cannot balance latency vs throughput dynamically

---

#### 2. Always Sync Per Write

- Immediate fsync

**Rejected because:**

- Severe throughput limitation
- Does not scale

---

#### 3. Fully Asynchronous Sync

- No immediate durability

**Rejected because:**

- Violates durability guarantees
- Risks data loss for acknowledged writes

---

### Rationale

Adaptive Sync enables:

```text id="2v5x9a"
Dynamic balancing of latency and throughput
```

While preserving:

```java
Strict durability guarantees and deterministic recovery
```

It leverages:

- Group Commit → batching mechanism
- WAL Buffer → staging capacity

To optimize fsync behavior under varying workloads.

---

### Notes

- Adaptive Sync is part of **Layer C (Durability Policy)**
- It modifies **policy execution**, not **policy definition**
- It is orthogonal to:

  - WAL structure
  - Record format
  - Lifecycle

---

### Invariants Introduced

- Adaptive sync must never violate durability boundary
- ACK contract remains unchanged regardless of policy adaptation
- WAL ordering and commit group ordering must be preserved
- All durability decisions must remain deterministic for a given execution

---

### Summary

Adaptive Sync transforms:

```java
Static durability timing → Workload-aware durability timing
```

Without compromising:

```java
Correctness, durability guarantees, or recovery semantics
```
