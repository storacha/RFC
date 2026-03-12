# RFC: Space Diff Deduplication

## Authors

- [Natalie Bravo](https://github.com/bravonatalie), [Storacha Network](https://storacha.network/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Introduction

The current `space-diff` table has accumulated structural issues that impact billing and system reliability due to duplicate entries. This RFC proposes changes to prevent duplicate diffs and simplify long-term maintenance.

### Problem Statement

Past bugs caused multiple diffs to be written for the same cause (e.g. failed uploads). This resulted in duplicated diffs that inflate usage, slow down queries and create "ghost" usage for spaces that should be empty after deletion.

This behavior should be **structurally impossible** going forward.

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

## Proposal

### Short-term solution: Deduplicate on write using a GSI

Add a **GSI on `cause`** to the existing `space-diff` table. Before inserting a new diff, query the GSI to check whether a diff with the same `cause` already exists. If it does, skip the write.

This approach:

* Prevents new duplicates without changing the table schema
* Can be deployed quickly with minimal risk
* Does not require a migration or dual-write strategy

**Limitation:** This is a best-effort guard — it adds a read-before-write cost and does not structurally prevent duplicates (a race condition is still theoretically possible).

### Medium-term solution: TTL-based archival to Glacier

Add a **TTL attribute** to the `space-diff` table so that items older than 1 year are automatically expired by DynamoDB. Before expiration, use a **DynamoDB Streams + Lambda** pipeline to archive expired items to S3 Glacier.

This approach:

* Keeps the table lean over time, improving query performance
* Reduces storage costs for historical data
* Supports retention policies without manual cleanup

### Long-term solution: New table with structural uniqueness

To guarantee uniqueness and prevent future duplication at the schema level:

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
3. Cut over later: switch usage reporting and billing reads to the new table; keep the existing table as read-only historical storage.
