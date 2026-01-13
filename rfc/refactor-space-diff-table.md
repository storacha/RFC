# RFC: Space Diff Refactoring

## Authors

- [Natalie Bravo](https://github.com/bravonatalie), [Storacha Network](https://storacha.network/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Introduction

The current `space-diff` table has accumulated some structural and operational issues that impact billing, usage calculation, and system reliability. This RFC proposes structural changes to make usage calculation efficient, prevent duplicate diffs, and simplify long-term maintenance.

### Problem Statement

### 1. Duplicate space diffs

Past bugs caused multiple diffs to be written for the same cause (e.g. failed uploads). This resulted in duplicated diffs that inflate usage, slow down queries and create “ghost” usage for spaces that should be empty after deletion.

This behavior should be **structurally impossible** going forward.

### 2. Usage calculation timeouts

A single space can generate a very large number of diff entries within the current month. When this happens, usage record calculation often times out because the system needs to aggregate too many records.

**Current mitigation (temporary):**

* A *space diff compaction* script that:
  * Aggregates many diffs into a single “summary” diff.
  * Archives the original diffs into a separate table.

This is an ad-hoc workaround and not a long-term solution.

## Current `space-diff` usage model

The `space-diff` table is the single **source of truth** for billing. It is written to by different sources depending on the protocol.

### Source A: Modern Blob Protocol

```
blob/accept OR blob/remove → blob-registry.register()
    1. allocation table entry (legacy compatibility)
    → TransactWrite {
        2. blob-registry table entry (primary storage)
        3. space-diff table entry (billing)
    }

```

- **Location**: `upload-api/stores/blob-registry.js`

### Source B: Legacy Store Protocol

Deprecated, but still operational for existing clients.

```
store/add OR store/remove receipt → UCAN stream → ucan-stream-handler → space-diff table

```

- **Location**: `billing/functions/ucan-stream.js`

### How usage is calculated today

This flow is used during billing runs for each space:

**Initial state**

* Load the space snapshot from `space-snapshot` for the `from` date
* If no snapshot exists, assume the space was empty (`size = 0`)

**Usage calculation**

* Base usage = `initialSize × periodDurationMs`
* Fetch all space diffs for the billing period
* Iterate diffs in chronological order:
  * `size += diff.delta`
  * `usage += size × timeSinceLastChange`
    * where `timeSinceLastChange = diff.receiptAt - lastReceiptAt`

**Storage**

* Store final space size in `space-snapshot` with `recordedAt = to`
* Store total usage in `usage` (byte-milliseconds)

## Proposal

### Fix for problem 1: Duplicate diffs

To guarantee uniqueness and prevent future duplication:

* Use **`cause` as the sort key (SK)** of the `space-diff` table
* This makes it impossible to insert two diffs for the same `(space, cause)` pair

#### Open design concern

Using `cause` as the SK removes natural chronological ordering.

**Proposed solution**

* Add a **GSI with a timestamp-based sort key**

This enables:

* Efficient chronological queries
* Time-based pagination
* Retention policies (e.g. deleting data older than 1 year)

The additional cost is acceptable, especially since older diffs can be safely deleted after the retention window.

#### Migration plan (high level)

1. Create a **new `space-diff` table** with:
   * Correct PK design
   * `cause` as SK
   * GSI for timestamp-based queries
2. Export data from the existing table
3. Deduplicate and transform records
4. Import data into the new table
5. Update application code to use the new schema
6. Decommission the old table after validation is complete

### Fix for problem 2: Usage calculation timeouts

Introduce a new table (e.g. `space-usage-month`) keyed by `provider#space#YYYY-MM` that is updated atomically on each diff write, making billing reads **O(1)**.

#### Core idea

Maintain a running usage accumulator instead of scanning historical diffs.

**Algorithm**

1. Track `lastSize` and `lastChangeAt` per `(provider, space, month)`
2. On each incoming diff:
   * `usage += lastSize × (receiptAt - lastChangeAt)`
   * `lastSize += delta`
   * `lastChangeAt = receiptAt`
3. At end-of-month billing:
   * `usage += lastSize × (periodEnd - lastChangeAt)`
   * Finalize and snapshot

**Additional fields**

* `sizeStart`
* `sizeEnd`
* `lastReceiptAt`
* `subscription` 

**Behavior**

* `space-diff` remains for audit and idempotency
* Billing reads exclusively from `space-usage-month`
* `calculatePeriodUsage`:
  * First tries the aggregator
  * Falls back to a GSI scan if missing
* Aggregator becomes the canonical source for the billing month

**Retention**

* Keep `space-diff` entries for N months using TTL
* Archive older diffs to S3 (TBD)

**Considerations**

- The accumulator MUST process diffs for a space in **ascending** `receiptAt` order. If the write path can deliver out-of-order events and strict ordering cannot be guaranteed, this solution SHOULD be revisited. Pragmatic mitigations include:
  - Buffer within a small window and sort incoming diffs.
  - Recompute a localized suffix by reading recent diffs via the time GSI and re-applying from the last stable checkpoint.

- Alternative when strict ordering is infeasible:
  - Use time-bucketed diffs (hour/day): persist per-bucket, order-independent aggregates (e.g., Σdelta and Σ(delta × (bucketEnd − receiptAt))). At billing time, iterate buckets in chronological order to compute exact monthly usage, where no event sorting required.
  - Maintain a size-only monthly state (track `lastSize` and `lastChangeAt`) to accelerate space usage report. Note: this does NOT remove the need to iterate diffs for the billing run.
