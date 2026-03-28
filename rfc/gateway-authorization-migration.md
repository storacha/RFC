# RFC: Gateway Authorization for Legacy Spaces Migration
Status: Experimental

## Authors

- [Felipe Forbeck](https://github.com/fforbeck), [Storacha Network](https://storacha.network/)

## Abstract

This RFC proposes a simplified authorization mechanism for migrating legacy spaces to the new Storacha gateway infrastructure. Legacy spaces lack private keys and cannot create valid UCAN delegations, requiring a special authorization flow. The proposal is that the gateway creates and signs delegations on behalf of legacy spaces, then trusts its own signatures without complex attestation validation.

## Introduction

### Background

The Storacha platform is migrating legacy spaces (created before the current UCAN-based authorization system) to work with the new gateway infrastructure. These legacy spaces have a fundamental limitation: they have no private keys and cannot create cryptographically valid UCAN delegations.

### Problem Statement

Initially, I attempted to solve this using UCAN attestations:
1. Create an Absentee delegation (space → gateway) with no valid signature
2. Have the gateway attest to the Absentee delegation using `ucan/attest`
3. Validate the attestation using validator proofs

This approach fails because:
- The `claim()` or `access()` validation function processes all capabilities in the proof chain, including `ucan/attest`
- The capability parser for `space/content/serve` doesn't recognize `ucan/attest` and flags it as an "unknown capability"
- Even with validator proofs configured, the capability filtering happens before attestation validation
- The UCAN validation flow is designed for attestations from external authorities, not self-attestations
- So I tested the attestation from the Upload Service to the Gateway, but it failed with the same error - even after adding the validator proofs saying that the gateway is authorized to attest to the delegation from the Upload Service

### Proposed Solution

Instead of using attestations, the idea is that the gateway creates simple self-signed delegations and trusts its own signatures without validation. This leverages the fact that the gateway's private key is the ultimate source of authority for what content it chooses to serve.

## Specification

### Migration Flow

During legacy space migration, the migration service creates a gateway-signed delegation:

```javascript
const contentDelegation = await SpaceCapabilities.contentServe.delegate({
  issuer: gatewaySigner,           // Gateway signs the delegation
  audience: gatewayPrincipal,      // Gateway is the audience
  with: space,                     // For the legacy space
  nb: {},                          // No additional constraints
  proofs: [],                      // No proofs needed - self-authorized
  expiration: Infinity,
})
```

This delegation is then published to the gateway via the `access/delegate` invocation.

### Gateway Handler Changes

The gateway's `access/delegate` handler is modified to detect and trust self-signed delegations:

**File**: `/freeway/src/server/service.js`

```javascript
import { UCAN } from '@ucanto/core'
import { Failure } from '@ucanto/validator'

export function createService (ctx, env) {
  return {
    access: {
      delegate: async (invocation, context) => {
        const capability = invocation.capabilities[0]
        
        try {
          const result = await extractContentServeDelegations(capability, invocation)
          if (result.error) {
            return result
          }

          const delegations = result.ok
          const validationResults = await Promise.all(
            delegations.map(async (delegation) => {
              // Check if this is a gateway self-signed delegation
              const isGatewaySelfSigned = 
                delegation.issuer.did() === ctx.gatewayIdentity.did() &&
                delegation.audience.did() === ctx.gatewayIdentity.did()

              if (isGatewaySelfSigned) {
                // Gateway self-signed delegation - verify signature explicitly
                // since we're bypassing claim() validation
                console.log('Gateway self-signed delegation - verifying signature')
                
                const signatureValid = await UCAN.verifySignature(
                  delegation.data,
                  ctx.gatewayIdentity
                )
                
                if (!signatureValid) {
                  return error(new Failure('Invalid signature on gateway self-signed delegation'))
                }
                
                console.log('Signature valid - storing delegation')
                const space = capability.with
                return ctx.delegationsStorage.store(space, delegation)
              } else {
                // External delegation - validate with claim()
                console.log('External delegation - validating with claim()')
                const validationResult = await claim(
                  SpaceCapabilities.contentServe,
                  [delegation],
                  {
                    ...context,
                    authority: ctx.gatewayIdentity,
                  }
                )
                if (validationResult.error) {
                  return validationResult
                }
                const space = capability.with
                return ctx.delegationsStorage.store(space, delegation)
              }
            })
          )

          const errorResult = validationResults.find((result) => result.error)
          if (errorResult) {
            return errorResult
          }

          return ok({})
        } catch (err) {
          return error(new Failure('error while processing authorization request', { cause: err }))
        }
      }
    }
  }
}
```

### Authorization Middleware Changes

The `withAuthorizedSpace` middleware is modified to verify and authorize gateway self-signed delegations:

**File**: `/freeway/src/middleware/withAuthorizedSpace.js`

```javascript
import { UCAN } from '@ucanto/core'

async function authorize(space, ctx, validatorProofs) {
  const delegationProofs = await ctx.delegationsStorage.find(space)
  
  if (delegationProofs.length === 0) {
    return { error: new Unauthorized(`no delegation found for space: ${space}`) }
  }

  // Check if this is a gateway self-signed delegation
  const delegation = delegationProofs[0]
  const isGatewaySelfSigned = 
    delegation.issuer.did() === ctx.gatewayIdentity.did() &&
    delegation.audience.did() === ctx.gatewayIdentity.did()

  if (isGatewaySelfSigned) {
    // Gateway self-signed delegation - verify signature before authorizing
    // This is a second verification point to ensure the delegation wasn't
    // tampered with after storage
    const signatureValid = await UCAN.verifySignature(
      delegation.data,
      ctx.gatewayIdentity
    )
    
    if (!signatureValid) {
      return { error: new Unauthorized('Invalid signature on gateway self-signed delegation') }
    }
    
    // Signature valid - authorize immediately
    return { ok: { delegation } }
  }

  // External delegation - validate with access()
  const accessResult = await access(
    {
      capability: SpaceCapabilities.contentServe.create({
        with: space,
        nb: {},
      }),
      authority: ctx.gatewayIdentity,
      proofs: delegationProofs,
    },
    {
      ...ctx,
      proofs: validatorProofs,
    }
  )

  if (accessResult.error) {
    return accessResult
  }

  return { ok: { delegation: delegationProofs[0] } }
}
```

## Security

### Signature Verification

Gateway self-signed delegations are protected by cryptographic signature verification at two critical points:

1. When the delegation is stored in delegations storage
2. When the delegation is authorized in the gateway

### How Signature Verification Works

The `UCAN.verifySignature()` function (from `@ipld/dag-ucan`):
1. Extracts the issuer's DID from the delegation
2. Derives the public key from the DID
3. Verifies the delegation's cryptographic signature using that public key
4. Returns `true` only if the signature is valid

### Trust Model

The security of this approach relies on:
1. **Gateway Private Key Protection**: Stored in Cloudflare Workers secrets (encrypted at rest)
2. **Controlled Delegation Creation**: Only the migration service can trigger gateway-signed delegation creation
3. **Signature Verification**: Cryptographic proof that delegations are authentic
4. **Dual Verification**: Signatures checked both at storage and authorization time
