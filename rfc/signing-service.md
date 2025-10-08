# Authorizing Proof Data Provider Operations

## Problem

Node providers on the Storacha network need to perform operations on proof sets - adding piece roots, deleting roots, and other state changes. 
These operations require signatures from Storacha to be valid on-chain.

Currently, we don't have a clean way to authorize these operations. Node providers need to:
1. Prove they're authorized to request signatures from Storacha
2. Get Storacha's signature for their operation payload
3. Use that signature to submit blockchain transactions

We need a system that:
- Only allows authorized operators to request signatures
- Prevents unauthorized access to the signing service
- Maintains a verifiable chain of authority back to Storacha
- Doesn't require manual management of operator allow-lists
- Uses short-lived delegations instead of relying on revocations

## Background

### EIP-712: Typed Structured Data Signing

The operations we're authorizing need to be submitted as blockchain transactions. 
Specifically, we're calling smart contract methods like `addPieces()` and `deletePieces()` on Filecoin.

These contract methods require signatures that prove the operations were authorized by Storacha. 
We use [EIP-712](https://eips.ethereum.org/EIPS/eip-712) for this signing.

EIP-712 is an Ethereum standard for signing typed, structured data. 
Instead of signing raw bytes (which are hard for users to verify), EIP-712 lets you sign structured data with clear types and fields. 
The signature proves:
1. The exact data being signed (operation type, parameters, values)
2. Which contract the data is intended for (domain separation)
3. Who signed it (via signature verification on-chain)

For example, an `AddPieces` operation might have a structure like:

```solidity
struct AddPieces {
  uint256 clientDataSetId;
  uint256 firstAdded;
  bytes32[] pieceData;
  bytes[] metadata;
}
```

The EIP-712 signature of this structure can be verified on-chain by the smart contract, which checks that it was signed by Storacha's authorized key.

This is why we need a signing service. 
The private key that creates these EIP-712 signatures must be controlled by Storacha, but node providers need to be able to request signatures for their operations. 
The signing service is the bridge - it holds Storacha's key and issues signatures to authorized operators.

## Proposal

### UCAN-Authorized Signing Service

We'll create a signing service that uses UCAN delegations to authorize signature requests. 
The flow works like this:

1. Storacha issues delegations to authorized operators granting them the `pdp/sign` capability
2. Operators create UCAN invocations requesting signatures, proving their authority via the delegation chain
3. The signing service validates the delegation chain and signs the operation payload
4. Operators use the returned signature for blockchain transactions

The key insight: the delegation chain IS the authorization. 
If an operator has a valid delegation from Storacha, they can request signatures. 
No separate allow-list needed.

### The `pdp/sign` Capability

We'll introduce a new capability `pdp/sign` that represents the ability to request signatures for proof data provider operations.

Storacha has the natural authority to grant this capability, as it controls the signing keys used for on-chain operations.

### Flow

#### Step 1: Operator Creates UCAN Invocation

When a node provider needs to perform an operation (like adding pieces), they create a UCAN invocation:

```
┌─────────────────────────────────────────┐
│ Issuer:    did:key:PiriNode...          │ ← Operator's identity
│ Audience:  did:key:SigningService       │ ← Signing service identity
│ Capability:                             │
│   with: "did:key:SigningService"        │ ← Resource (signing service)
│   can:  "pdp/sign"                      │ ← Ability
│   nb:   {                               │ ← Caveats
│     operation: "AddPieces",             │
│     payload: {                          │
│       clientDataSetId: "123",           │
│       firstAdded: "0",                  │
│       pieceData: ["0xabc...", ...],     │
│       metadata: [...]                   │
│     }                                   │
│   }                                     │
│ Proofs: [                               │ ← Delegation chain
│   <delegation from Storacha>            │   (proves operator can pdp/sign)
│ ]                                       │
│ Signature: <signed by operator key>     │
└─────────────────────────────────────────┘
```

The `proofs` field contains a delegation from Storacha that looks like:

```
┌─────────────────────────────────────────┐
│ Issuer:    did:web:storacha.network     │ ← Storacha's identity
│ Audience:  did:key:PiriNode...          │ ← Operator's identity
│ Capability:                             │
│   with: "did:key:SigningService"        │
│   can:  "pdp/sign"                      │
│   nb:   {                               │
│     operations: [                       │
│       "AddPieces",                      │
│       "DeletePieces",                   │
│       ...                               │
│     ]                                   │
│   }                                     │
│ Expiration: <short lived, e.g. 1 day>   │
│ Signature: <signed by Storacha>         │
└─────────────────────────────────────────┘
```

#### Step 2: Send to Signing Service

```
Piri Node
  ↓
  HTTP POST to signing-service.storacha.network
  Content-Type: application/car
  Body: <CAR file containing UCAN invocation>
  ↓
Signing Service
```

#### Step 3: Signing Service Validates UCAN

The signing service validates the invocation:

1. **Verify UCAN signature**
   - Is the signature valid for the issuer DID?

2. **Check delegation chain**
   - Do the proofs link back to Storacha's root authority?
   - Is each delegation in the chain valid and not expired?
   - Does the chain grant the `pdp/sign` capability?

3. **Verify audience**
   - Is the audience the signing service's DID?

4. **Validate caveats**
   - Is the operation valid (e.g., "AddPieces", "DeletePieces")?
   - Does the payload have all required fields?

If validation fails, return a UCAN receipt with an error. 
If it succeeds, proceed to signing.

#### Step 4: Signing Service Signs Payload

The signing service extracts the operation and payload from the caveats and calls the appropriate EIP-712 signer:

```go
operation := invocation.Capability.Nb["operation"]
payload := invocation.Capability.Nb["payload"]

// For AddPieces 'operation':
signature := signer.SignAddPieces(
  payload.ClientDataSetId,
  payload.FirstAdded,
  payload.PieceData,
  payload.Metadata,
)

// Returns: {v: 28, r: 0x..., s: 0x..., signature: 0x...}
```

#### Step 5: Signing Service Returns UCAN Receipt

The signing service creates a receipt linking back to the invocation:

```
┌─────────────────────────────────────────┐
│ Issuer:  did:key:SigningService         │ ← Signing service identity
│ Ran:     <CID of invocation>            │ ← Links to the request
│ Out:                                    │
│   ok: {                                 │
│     v: 28,                              │
│     r: "0x1234...",                     │
│     s: "0x5678...",                     │
│     signature: "0xabcd...",             │
│     signedData: "0xef01...",            │
│     signer: "0x742d35..."               │
│   }                                     │
│ Signature: <signed by signing service>  │
└─────────────────────────────────────────┘
```

For errors:

```
┌─────────────────────────────────────────┐
│ Issuer:  did:key:SigningService         │
│ Ran:     <CID of invocation>            │
│ Out:                                    │
│   error: {                              │
│     name: "UnauthorizedError",          │
│     message: "Invalid delegation chain" │
│   }                                     │
│ Signature: <signed by signing service>  │
└─────────────────────────────────────────┘
```

#### Step 6: Operator Receives Response

```
Signing Service
  ↓
  HTTP 200 OK
  Content-Type: application/car
  Body: <CAR file containing UCAN receipt>
  ↓
Piri Node
```

The operator validates the receipt and extracts the signature:

```go
// Validate receipt
if receipt.Ran != invocationCID {
  return errors.New("receipt doesn't match invocation")
}

// Verify receipt signature
if !verifySignature(receipt, signingServiceDID) {
  return errors.New("invalid receipt signature")
}

// Extract signature components
result := receipt.Out.Ok
v := result.V
r := result.R
s := result.S

// Submit blockchain transaction
contract.AddPieces(v, r, s, clientDataSetId, firstAdded, pieceData, metadata)
```

### Benefits

**No manual operator management**: The delegation chain proves authorization. No need to maintain allow-lists in the signing service.

**Short-lived delegations**: Storacha can issue delegations with short expiration times (e.g., 1 day). Revocations are rarely needed.

**Auditable**: Every signature request is a UCAN invocation with a full delegation chain. We can trace who requested what and when.

**Flexible**: Easy to add new operations or adjust scopes in delegations without redeploying services.

**Standard**: Uses UCAN patterns consistent with the rest of the network.

### Implementation Notes

**Delegation distribution**: Storacha needs a mechanism to issue delegations to new operators. This could be:
- Automated when operators join the network
- Requested via an `access/authorize` flow (see https://github.com/storacha/RFC/pull/68)
- Issued through an admin interface

**Delegation caching**: Operators should cache delegations until they expire to avoid requesting new ones for every operation.

**Replay protection**: The blockchain transaction layer handles replay protection via nonces. 
The signing service doesn't need to track used invocations, though it could for monitoring purposes.

**Operation types**: Initial operations will be `CreateDataSet`, `AddPieces` and `DeletePieces`. 
The design supports adding more operation types as needed.

