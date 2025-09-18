# Account Usage Get

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- Storacha team
- Natalie Bravo

## Introduction

The `usage/report` capability enables an agent to obtain storage usage information on a space. 

Currently, whenever a user wants to view their total storage usage, whether via CLI or console, we can only provide this information if the agent being used has permission to access every space. In practice, this often isn’t the case<sup>[1](#note1)</sup>, which leads to a misalignment between the current definition of this capability and how it’s actually used across our system.

With Storacha’s growing adoption through other tools, such as the Telegram Mini App and Bluesky, the total usage view has become even more fragmented.

This RFC proposes a discussion on better approaches.


<a id="note1"></a>
> <sub>**Note 1:** Scenarios where we don't have the total view of the users spaces: 
> - If a user has lost the private key for a space and didn’t delegate control to us.  
> - If a user didn’t delegate control to us and is using a device without the appropriate delegation from a space. 
> - If a user logs in to a new agent (B) but creates a new space (2) on the old agent (A), agent B wouldn’t know about space 2.</sub>


## Proposal

A new capability that can be invoked in the context of an Account DID (aggregating usage across all spaces). 

### `account-usage/get`

#### Invocation

```ipldsch
type AccountUsageGet struct {
  with AccountDID
  nb optional AccountUsageGetNB
}

type AccountUsageGetNB struct {
  space optional [SpaceDID]

  # Optional time period (Unix timestamps in seconds).
  # If omitted, provider MUST return a current snapshot.
  period optional Period
}

type Period struct {
  from Int # inclusive
  to Int # inclusive
}
```

> example: getting the current total usage

```json
{
  "iss": "did:key:z6MktfnQz8Kcz5nsC65oyXWFXhbbAZQavjg6LYuHOOOagent",
  "aud": "did:web:storacha.network",
  "att": [
    {
      "with": "did:mailto:web.mail:alice",
      "can": "account-usage/get"
    }
  ],
  "prf": [
    { "/": "bafyAccountDelegationCid" }, 
    { "/": "bafySessionAttestationCid" }
  ],
  "sig": "..."
}
````

> example: filtering a single space and period

```json
{
  "iss": "did:key:z6MktfnQz8Kcz5nsC65oyXWFXhbbAZQavjg6LYuHOOOagent",
  "aud": "did:web:storacha.network",
  "att": [
    {
      "with": "did:mailto:web.mail:alice",
      "can": "account-usage/get",
      "nb": {
          "spaces": ["did:key:z6MkuxVKbEvYzXw89c9ESd3xoZ988MFrCgqT5JF5wtBvuYWe"],
          "period": {
            "to": 1758111728,
            "from": 1754006400
          },
      }
    }
  ],
 "prf": [
    { "/": "bafyAccountDelegationCid" }, 
    { "/": "bafySessionAttestationCid" }
  ],
  "sig": "..."
}
```

#### Usage Calculation

**Total usage** is the aggregate usage (e.g., total bytes stored) across all spaces for the requested Account DID.

If `period` is omitted; for each space, usage is computed as:

- The most recent snapshot available (typically the beginning of the current billing cycle, e.g., `startOfLastMonth(now)`).

- Plus the sum of all space diff events (deltas) up to the current time (`to`) or "now".

> **Note:**
> To guarantee reproducible results (idempotency), clients SHOULD specify an explicit `period` when querying.

#### Receipt

```ipldsch

type AccountUsageGetReceipt = {
  ran: Link<AccountUsageGet>
  out: Result<AccountUsageGetOk, AccountUsageGetError>
}

type AccountUsageGetError {
    message: string
}

type AccountUsageGetOk {
    total        Int
    spaces       {String: SpaceUsage}   # key: SpaceDID, keys MUST be sorted
}

type SpaceUsage {
  total     Int
  providers {String: ProviderUsage}    # key: ProviderDID, keys MUST be sorted
}

type ProviderUsage {
  provider ProviderDID
  space    SpaceDID
  period   PeriodISO
  size SizeDelta
  events optional [UsageEvent]
}

type SizeDelta {
  initial Int
  final   Int
}

type UsageEvent {
  cause     Link
  delta     Int
  receiptAt ISO8601Date
}

type PeriodISO {
  from ISO8601Date
  to   ISO8601Date
}

type ISO8601Date = string
type ProviderDID = string
type DID = string

```

In all responses, the keys of the `spaces` field in `AccountUsageGetOk` and the `providers` field in `SpaceUsage` **MUST be sorted lexicographically** by their respective key (SpaceDID, ProviderDID). This ensures that the same query produces the same output each time.


> example:

```json
{
  "total": 5356848797,
  "spaces": {
    "did:key:z6MkuxVKbEvYzXw89c9ESd3xoZ988MFrCgqT5JF5wtBvuYWe": {
      "total": 5356848797,
      "providers": {
        "did:web:web3.storage": {
          "provider": "did:web:web3.storage",
          "space": "did:key:z6MkuxVKbEvYzXw89c9ESd3xoZ988MFrCgqT5JF5wtBvuYWe",
          "period": {
            "to": "2025-08-12T15:55:11.000Z",
            "from": "2025-07-01T00:00:00.000Z"
          },
          "size": {
            "final": 5356848797,
            "initial": 1035217049
          },
          "events": [
            {
              "cause": {
                "/": "bafyreiafouslzry3vunc4okazrqpwpuanrq4d3ehb2oatn4vmrhnmj6rzy"
              },
              "delta": 54394,
              "receiptAt": "2025-07-01T14:34:50.947Z"
            }
          ]
        }
      }
    }
  }
}

```

Obs.: The current `usage/report` returns the `ProviderUsage` as a receipt.


## Implementation

This plan outlines the changes needed to create a new usage capability

### 1. Specification Definition
Create a formal capability specification in the `specs` repository. (The current `usage/report` capability lacks a formal spec) 
 
### 2. Capability definition
Add the new capability definition in on `upload-service/packages/capabilities`. Also create new types to support it.

### 3. Client implementation
Update the `UsageClient` to support the new capability to on `upload-service/packages/w3up-client/src/capability`.

### 4. Service implementation
This involves two parts:

- **Upload API Handler**: Add a new handler in `upload-service/packages/upload-api` to process the new capability.
- **Storage Layer**: Update `w3infra/upload-api/stores/usage.js` to handle the new capability.


### 5. Integrations

- **CLI**: Add a new command on the CLI to support the new usage capability.
- **Console**: Replace the current `usage/report` invocations in the console application with calls to the new capability.