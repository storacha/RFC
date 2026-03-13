# RFC: Billing Dashboard

## Authors

- [Vicente Olmedo](https://github.com/volmedo), [Storacha Network](https://storacha.network/)

## Introduction

We need to offer a window the different actors in the Storacha Forge Network can use to get the metrics and stats they need. The etracker/billing service will be offering such window in the form of a dashboard, which will provide different information to different roles in the network.

## Personas

### Customers (aka Clients)

Users that pay for a (reserved) storage capacity and the egress of that data from their spaces.

### Node operators

These users add storage capacity to the network by provisioning and maintaining piri (storage) nodes. They get paid for the amount of data they store and also for the data egressed from their nodes. A single node operator can operate more than one piri node.

### Storacha (aka Admins)

We (the Storacha team) need visibility on billing metrics so that we can charge customers for storage capacity and egress, and compensate node operators for the storage and egress provided by the nodes they operate.

## Current state: admin dashboard

There is an initial implementation of the admin dashboard that satisfies the needs of the Storacha team as admins of the network. This initial implementation focuses on:

- **Single metric**: Egressed bytes only (no other billing dimensions)
- **Fixed periods**: Previous month, current month, current week, current day. No ability to configure periods or thresholds (future enhancement)
- **Node-level stats**: Total egress for the node (not broken down by space for now)

The admin dashboard was implemented as a template rendered server-side with pre-configured credentials for a single admin user as the fastest way to deliver value.

## Customer dashboard

The next step in the billing dashboard implementation will be to deliver a customer view. The [Hybrid Approach](#alternative-3-hybrid-approach) will be used to make it consistent with how the hot storage console application works.

### Requirements

- **Accounting considerations**:
  - Invoices (calendar monthly): as a client of the service I can see egress fees and storage fees for the calendar month (amount in TiB as well as total price). Example:

  ```
    Storage Use for month: 5 TiB
    Price of Storage: 5 TiB x 5.99 per TiB per Month
    Egress for the month: 2 TiB
    Price for Egress: 2 TiB x 10 per TiB (in the calendar month)
  ```

  - We don’t need the dashboard to produce these invoices, just allow for the data to be displayed in this manner so we can produce the invoices manually AND the customer can see how much they owe and why.

- **Monitoring considerations**:
  1. As a client of the service, I can see how much data has been stored in my account (daily)
  1. As a client of the service, I can see how much data has been served from my account (daily)
  1. As a client, I can see how much storage capacity I have reserved, used, and remaining

### Implementation details

- Implement the Storacha Forge console application equivalent. This implies using UCANs for auth and APIs. Authentication will be done via Storacha's email login flow, same as the hot network's console app.

  Note that this approach is different from the one used for the admin dashboard, which is not ideal, but they serve different purposes after all. We may consider merging the two later.

- Daily egress data is readily available from the etracker/billing service (monitoring requirement 2). A new capability will be added to the etracker/billing service to expose this data.
- Storage data is available from the upload service. However, we need to take care of the following:
  - We need the storage capacity reserved by the customer in order to fulfill the monitoring requirement 3. In the hot network, storage capacity is not an arbitrary number. It is given by the subscribed plan. In the forge network, however, clients can reserve any amount of storage capacity they want, in multiples of 1 TiB. The proposal is to add a new attribute (which will be populated manually for now) to the upload service's customers table. It is likely that plan information is not meaningful in the forge network.
  - Monitoring requirement 1 requires daily storage data. The upload service exposes this data via the `account/usage/get` capability. It will require some client-side logic to roll up the data into daily totals.

### Proposal

- Daily egress data via a new `account/egress/get` capability in the etracker/billing service.
- Daily storage data via the existing `account/usage/get` capability in the upload service.
- New attribute in the upload service's customers table to store the storage capacity reserved by the customer.
- Do not break down by space for now. The data is there, so it should be easy to add later.

## Future Considerations

- **Custom date ranges**: Allow specifying arbitrary time periods
- **Space-level granularity**: Break down stats by individual spaces
- **Additional metrics**: Request counts, earnings
- **CSV export**: Enable data export for analysis

## Appendix A: Implementation Alternatives

### Alternative 1: Web Application

Implement a traditional web application where the etracker/billing service provides both an HTTP API and a frontend UI.

**Architecture**:

```
┌─────────────┐
│   Browser   │
└──────┬──────┘
       │ HTTPS (username/password auth)
       ↓
┌─────────────────────────────┐
│   Billing Service           │
│   - Frontend (React/Vue)    │
│   - REST API                │
│   - User auth system        │
└─────────────────────────────┘
```

**Authentication Flow**:

1. Operator creates account with email/password (or OAuth)
2. Operator logs in and receives session token/JWT
3. Operator associates nodes with their account by proving ownership
4. Frontend makes authenticated API requests with session token

**Node Ownership Verification**:
Operators must prove they control a node (identified by its DID). Options include:

- **Challenge-response**: Service issues a challenge, operator signs with node key, submits signature
- **Delegation proof**: Operator provides a UCAN delegation from the node DID to their account
- **Pre-registration**: Node DID is linked to user account during onboarding

**✅ Pros**:

- **Rich UI**: Beautiful charts, graphs, and trend visualizations
- **Familiar UX**: Users are accustomed to username/password authentication
- **Accessible**: Works from any device with a browser, no installation needed
- **Discoverability**: Features are easy to explore through GUI navigation
- **Lower barrier**: Non-technical users can access stats without CLI knowledge

**❌ Cons**:

- **Centralized authentication**: Requires building user account system (passwords, sessions, resets)
- **Node ownership complexity**: Must implement additional mechanism to prove node ownership
- **Security surface**: Password storage, session management, CSRF protection required
- **Architectural mismatch**: Traditional web auth doesn't align with UCAN-based network
- **Operational burden**: User database management, security incidents, compliance

### Alternative 2: CLI Subcommand in Piri

Define a new `billing/stats` capability. Implement billing statistics as a subcommand in the piri CLI, with the etracker/billing service exposing a UCAN-based API.

**Architecture**:

```
┌─────────────────────┐
│   Piri CLI          │
│   - piri billing    │
└──────────┬──────────┘
           │ UCAN invocations
           ↓
┌─────────────────────────────┐
│   Billing Service           │
│   - UCAN API handler        │
│   - billing/stats           │
└─────────────────────────────┘
```

**Authentication Flow**:

1. Operator invokes piri CLI command
2. Piri creates UCAN invocation signed by node key
3. Invocation includes delegation proofs as needed
4. Service validates UCAN delegation chain
5. Service returns receipt with billing statistics

**Example UCAN Invocation**:

```json
{
  "iss": "did:key:zNodeOperator",
  "aud": "did:web:billing.storacha.network",
  "att": [
    {
      "with": "did:key:zStorageNode",
      "can": "billing/stats"
    }
  ],
  "prf": [],
  "sig": "..."
}
```

**Example Receipt**:

```json
{
  "ran": { "/": "bafy...statsInvocation" },
  "out": {
    "ok": {
      "previousMonth": {
        "bytes": 1234567890,
        "period": {
          "from": "2025-09-01T00:00:00Z",
          "to": "2025-09-30T23:59:59Z"
        }
      },
      "currentMonth": { "bytes": 987654321, "period": {...} },
      "currentWeek": { "bytes": 123456789, "period": {...} },
      "currentDay": { "bytes": 12345678, "period": {...} }
    }
  }
}
```

**CLI Usage**:

```bash
piri billing                    # Show egress stats
piri billing --format json      # JSON output for scripting
```

**✅ Pros**:

- **Decentralized authentication**: Uses existing UCAN delegations, no centralized user database
- **Automatic node ownership**: Invocation signed by node key proves ownership
- **Architectural consistency**: Aligned with how other parts of the system operate
- **Simple implementation**: Define capability → implement handler → add CLI command
- **Auditable**: Every request is a signed UCAN invocation

**❌ Cons**:

- **Limited visualization**: CLI output is text-based, charts are ASCII art
- **Less discoverable**: Features aren't as discoverable as GUI menus
- **Device-specific**: Tied to the device running piri

### Alternative 3: Hybrid Approach

Build a UCAN-based API that can be consumed by both CLI and web interfaces.

**Architecture**:

```
┌─────────────┐         ┌─────────────────┐
│   Browser   │         │   Piri CLI      │
└──────┬──────┘         └────────┬────────┘
       │                         │
       │ UCAN invocations        │ UCAN invocations
       ↓                         ↓
┌────────────────────────────────────────┐
│   Billing Service UCAN API             │
│   - billing/stats                      │
└────────────────────────────────────────┘
```

**Authentication Flow**:

1. Auth via web frontend could be similar to how console works
2. Auth via CLI is straightforward (UCAN invocation)

**Implementation Strategy**:

- Build UCAN API first (enables CLI immediately)
- Add static web frontend later that uses same UCAN API
- Web UI uses JavaScript UCAN libraries (@ucanto/client)
- No server-side session state required

**✅ Pros**:

- **Incremental delivery**: Ship CLI support first, add web UI later
- **Consistent API**: Both CLI and web use identical UCAN-based API
- **Decentralized auth**: Even web UI uses UCAN delegations
- **Flexible UX**: Operators choose CLI for automation, web for visualization

**❌ Cons**:

- **Two interfaces to maintain**: Must develop both CLI and web UI
