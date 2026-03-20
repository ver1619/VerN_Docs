# ADR-004 Asynchronous WAL Segment Preallocation

### Status

Accepted

---

### Context

WAL segments are rotated when:

- The current segment reaches its size limit, OR
- A manual rotation is triggered (e.g., shutdown)

During rotation:

```java
ACTIVE → SEALED → create new ACTIVE
```

If segment creation/allocation occurs synchronously:

- Write path is blocked during allocation
- Filesystem operations (file creation, allocation) introduce latency
- This results in **visible write latency spikes**

This violates a desirable property:

> WAL write latency should remain stable and predictable

---

### Decision

Introduce **Asynchronous Preallocation** of WAL segments.

---

### Definition

Asynchronous Preallocation allows the system to:

- Prepare future WAL segments **before they are needed**
- Decouple segment allocation from the critical write path

---

### Behavior

- While a segment is `ACTIVE`, the system may:

  - Create and prepare the next segment in advance

- When rotation occurs:

  - Preallocated segment becomes the new `ACTIVE` segment

---

### Placement in Lifecycle

```java
CREATED (preallocated) → ACTIVE → SEALED → OBSOLETE → DELETED
```

Preallocated segments exist in `CREATED` state until activated.

---

### Constraints

- Preallocated segments must not accept writes until they become `ACTIVE`
- Preallocation must not block WAL writer execution
- Segment numbering must remain strictly monotonic

---

### Durability Interaction

- Preallocation does **not imply durability**
- A preallocated segment:

  - Contains no valid WAL data
  - Has no recovery significance until used

Durability applies only after actual WAL append + fsync.

---

### Interaction with WAL Preallocation Semantics

- File space may be reserved in advance
- Unwritten regions remain **invalid WAL data**
- Recovery must ignore all preallocated but unwritten regions

---

### Consequences

#### Positive

- **Eliminates rotation latency spikes**

  - No blocking file creation during write

- **Improves write path smoothness**

  - Stable latency under sustained workloads

- **Better filesystem behavior**

  - Allows controlled allocation patterns

---

#### Negative

- **Slight increase in resource usage**

  - Additional preallocated files may exist

- **Background work required**

  - Allocation must be scheduled outside write path

- **Potential unused segments**

  - Preallocated segments may not be immediately used

---

### Alternatives Considered

#### 1. Synchronous Allocation

- Allocate new segment during rotation

**Rejected because:**

- Causes write latency spikes
- Blocks WAL writer

---

#### 2. No Preallocation

- Create segments only when needed

**Rejected because:**

- Same latency issues as synchronous allocation
- Poor write path stability

---

#### 3. Bulk Preallocation

- Allocate multiple segments upfront

**Rejected because:**

- Excessive resource usage
- Reduced flexibility

---

### Rationale

Asynchronous preallocation ensures:

```java
Write path remains non-blocking during segment transitions
```

While preserving:

```java
Lifecycle correctness and recovery guarantees
```

---

### Notes

- Integrated into **WAL Lifecycle (rotation policy)**

- Does not modify:

  - WAL structure
  - Record format
  - Durability semantics

- Works in conjunction with:

  - WAL Buffer → smooth write flow
  - Group Commit → efficient batching

---

### Invariants Introduced

- Only one segment may be `ACTIVE` at any time
- Preallocated segments must not contain valid WAL records
- Preallocation must not interfere with segment ordering
- Segment activation must follow lifecycle transition rules

---

## Summary

Asynchronous Preallocation transforms:

```java
Blocking segment allocation → Background preparation
```

Without affecting:

```java
WAL correctness, ordering, or durability guarantees
```
