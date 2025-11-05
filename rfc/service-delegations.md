# RFC: Service Delegations

## Authors

- [Travis Vachon](https://github.com/travis), [Storacha Network](https://storacha.network/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Introduction

We'd like to be able to create delegations from the "service principal" to arbitrary email addresses and store them with storacha.network using the [`access/delegate`](https://github.com/storacha/specs/blob/main/w3-access.md#access-delegate) invocation. This will allow us to delegate capabilities to administrator email addresses and build user interfaces to support the administration of our services.

As an example, this will allow administrators to use the [Storacha admin app](https://github.com/storacha/admin/) without any special delegation creation - administrators will simply log in with their email address and rely on the support for ["claiming"](https://github.com/storacha/specs/blob/main/w3-access.md#access-claim) delegations stored with Storacha that is built in to our standard authorization tools.

While this is technically possible today by creating a space with an arbitrary user account and delegating `access/delegate` on that space to the "service principal" we propose using the service principal's `did:key` public key as a space. This obviates the needs for additional proofs - since we need to load the service principal's private key into memory to create service delegations at all, using the principal's `did:key` as a space for this invocation is tautalogically possible.

## Proposal

The following code 1) creates a delegation of several capabilities on the "service" to a "delegatee" and 2) submits that delegation to the service to be stored:

```ts
const client = await create({ principal: servicePrincipal, store: new MemoryDriver() })
const delegation = await delegate({
  // create a delegation from the "service principal"
  issuer: servicePrincipal,
  // to an arbitrary email address
  audience: Absentee.from({ id: delegateeDidMailto }),
  // grant the capabilities needed for admin work
  capabilities: [
    { with: servicePrincipal.did(), can: 'customer/get' },
    { with: servicePrincipal.did(), can: 'consumer/get' },
    { with: servicePrincipal.did(), can: 'subscription/get' },
    { with: servicePrincipal.did(), can: 'rate-limit/*' },
    { with: servicePrincipal.did(), can: 'admin/*' },
  ],
  expiration: Math.floor(Date.now() / 1000) + (60 * EXPIRY)
})
const resultFromServer = await client.capability.access.delegate({
  // use the did:key of the service principal as the space to store the delegation in
  space: servicePrincipal.toDIDKey(),
  delegations: [delegation]
})
```

This code currently results in the following error in `resultFromServer`:

```json
{
  error: {
    name: 'InsufficientStorage',
    message: 'did:key:z6MkqdncRZ1wj8zxCTDUQ8CRT8NQWd63T7mZRvZUX8B7XDFi has no storage provider'
  }
}
```

This is because `access/delegate`'s "with" resource is a designed to be a space - since `access/delegate` is just a special flavor of data storage, it, like most capabilities in our system, is designed around spaces and in this case it fails because the space in question has not been "provisioned" to be able to storage data.

To fix this, we propose manually editing our databases to give did:key:z6MkqdncRZ1wj8zxCTDUQ8CRT8NQWd63T7mZRvZUX8B7XDFi a "storage provider". It will, of course, never be charged for data storage. In the future when we rotate service keys we will need to use a different space, provisioning it manually as well.

Alternatively, we could update our code to always treat the "service key space" as provisioned - since the service key will still be needed to sign any invocations that cause data to be stored, this should be safe to do.