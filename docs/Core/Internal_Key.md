# Internal Key

**An internal key must satisfy the following:**
- Maintain a **total, deterministic ordering** across all storage layers
- Encode versioning **(MVCC)** for each user key
- Support **snapshot-based visibility**
- Represent **delete operations via tombstones**
- Enable **correct and efficient merge during compaction**
- Allow **early termination during point lookup**
- Ensure **comparator consistency** across all components

---

## 1. Key Structure

### 1.1 Logical Layout

```go
| user_key | sequence_number | value_type |
```

- **`user_key`** → actual key provided by the user
- **`sequence_number`** → version identifier
- **`value_type`** → operation type (`PUT`, `DELETE`)

### 1.2 Binary Layout

```go
[user_key bytes]
[8-byte packed suffix]
```

- **The suffix encodes:**
  - sequence number
  - value type
- The encoding must preserve ordering semantics

---

## 2. Ordering Model

### 2.1 Comparator Definition

Internal keys are ordered as:

```java
(user_key ASC, sequence_number DESC, value_type DESC)
```

Comparator rules:

1. Compare `user_key` lexicographically (ascending)
2. If equal → compare `sequence_number` (descending)
3. If equal → compare `value_type` (descending)

---

### 2.2 Ordering Guarantees

- Newer versions of a key appear before older versions
- For identical (`user_key`, `sequence_number`):
  - `DELETE` must sort before `PUT`
- Ordering is:
  - total
  - stable
  - deterministic

---

### 2.3 Comparator Consistency

The comparator must be **identical across all components**:

- MemTable
- SSTable (index + data blocks)
- Iterators
- Compaction logic

No deviation is permitted.

---

## 3. Visibility Model

### 3.1 Snapshot Visibility Rule

A record is visible to a read at snapshot S if:

```java
record.sequence_number ≤ S
```

---

### 3.2 Lookup Semantics

Given ordered keys:

```java
A@105 PUT
A@100 PUT
A@90  DELETE
```

A lookup:

- Scans in order
- Returns the first visible record

---

### 3.3 Early Termination Guarantee

Due to ordering:

- The first visible record is always the **correct result**
- No further versions need to be examined

---

## 4. Delete Semantics

### 4.1 Representation

```java
(user_key, sequence_number, value_type=DELETE)
```

---

### 4.2 Read Behavior

- If the first visible record is `DELETE`:
 - The key is considered **non-existent**
- No immediate physical deletion occurs

---

### 4.3 Dominance Rule

For equal (`user_key, sequence_number`):

```java
DELETE > PUT
```
This ensures deterministic conflict resolution.

---

## 5. Version Lifecycle

### 5.1 Version Retention

Multiple versions of a key may coexist:

- Across MemTable and SSTables
- Across different LSM levels

---

### 5.2 Obsolete Version Definition

A version is considered obsolete if:

```java
sequence_number < earliest_snapshot_sequence
```

Such versions are not visible to any active reader.

---

### 5.3 Compaction Eligibility

Obsolete versions may be removed during compaction, subject to:

- Snapshot visibility constraints
- Presence of newer versions

---

## 6. Tombstone Lifecycle

### 6.1 Retention Requirement

A tombstone must be retained if:

- There exists any older version that may still be visible
- Older versions may exist in lower levels

---

### 6.2 Safe Removal Condition

A tombstone may be removed only if:

```java
No version with sequence_number < tombstone.sequence_number
is visible to any snapshot
AND
No older version exists in lower levels
```

---

### 6.3 Safety Guarantee
Tombstone removal must never allow resurrection of deleted data

---

## 7. Interaction Contracts

### 7.1 Read Path

- Must evaluate keys in comparator order
- Must apply snapshot visibility rule
- Must terminate at first visible record

---

### 7.2 Compaction

- Must preserve comparator ordering
- Must respect snapshot visibility boundaries
- Must enforce tombstone retention rules
- Must only drop versions proven obsolete

---

### 7.3 Storage Layers

All storage layers must:

- Store keys in Internal Key format
- Use identical comparator semantics
- Preserve ordering during transformations

---

## 8. Constraints

- Internal Key must be **fully self-contained**
- Ordering must not depend on external metadata
- Encoding must be:
  - deterministic
  - comparable without decoding full structure

---

### Invariants

- Internal key ordering is **total and deterministic**
- Comparator is **globally consistent across all components**
- Sequence numbers are **globally monotonically increasing**
- Visibility rule:

```java
record.sequence_number ≤ snapshot
```

- First visible record in order defines the **correct result**
- DELETE dominates PUT for equal (`user_key, sequence_number`)
- No obsolete version is visible to any snapshot
- Tombstones are only removed when **provably safe**
- No operation violates ordering semantics