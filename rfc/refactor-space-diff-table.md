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
2. Enable dual-writes: on each diff event, write to both the existing table and the new table. Keep all readers (usage, reporting, billing) pointed at the existing table during January.
3. Cut over in February: switch usage reporting and billing reads to the new table; keep the existing table as read-only historical storage.

### Fix for problem 2: Usage calculation timeouts

Generate snapshots more frequently.

One option is to move snapshot generation to a daily cadence. There are two possible approaches:

1. **Decouple snapshot generation from the billing cron**

  * Generate snapshots independently, without running the full billing pipeline

2. **Run the full billing process daily**

  * This would naturally produce more snapshots and also push usage reports to Stripe more frequently

Since both approaches are pretty similar and would need to iterate over all customers and spaces to generate snapshots anyway, the main extra work with running the full billing flow is the usage calculation, writing usage records, and reporting to Stripe.

Given the upside of reporting to Stripe more frequently and the fact that this is simpler than setting up separate infra just for snapshot generation, we’ll move forward with option two.