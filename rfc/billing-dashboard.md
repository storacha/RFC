# RFC: Billing Dashboard

## Authors

- [Vicente Olmedo](https://github.com/volmedo), [Storacha Network](https://storacha.network/)

## Introduction

Node operators in the warm storage network need visibility into their egress usage - how much data has been retrieved from their nodes over time. This information is critical for understanding costs and revenue, and also useful to plan capacity.

This RFC proposes different implementation approaches for exposing billing statistics to node operators, starting with a simple scope: bytes egressed over predefined time periods (previous month, current month, current week, current day).

The solution must address:
- **Authentication**: How operators prove their identity
- **Authorization**: How operators prove they control specific nodes
- **Data access**: How operators retrieve egress statistics from the etracker/billing service

## Scope

This initial implementation focuses on:
- **Single metric**: Egressed bytes only (no other billing dimensions)
- **Fixed periods**: Previous month, current month, current week, current day. No ability to configure periods or thresholds (future enhancement)
- **Node-level stats**: Total egress for the node (not broken down by space for now)

## Alternatives

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

## Proposal

**Start with Alternative 2 (CLI Subcommand)**, with the option to evolve to Alternative 3 (Hybrid) based on user feedback/needs.

**Rationale**:
1. ✅ **Fastest time to value**: CLI implementation is straightforward and leverages existing piri infrastructure
2. ✅ **Architectural alignment**: Maintains UCAN patterns throughout the entire stack
3. ✅ **Target audience**: Node operators are technical users already using piri CLI
4. ✅ **Incremental path**: Can add web UI later without changing API or authentication model
5. ✅ **Proven pattern**: Similar to how `usage/report` and other capabilities work today

## Future Considerations

- **Custom date ranges**: Allow specifying arbitrary time periods
- **Space-level granularity**: Break down stats by individual spaces
- **Additional metrics**: Storage usage, request counts, earnings
- **CSV export**: Enable data export for analysis
