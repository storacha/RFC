# Proposal: Single-DID Service Entry

## Authors

- [Petra Jaros](https://github.com/Peja), [Storacha Network](https://storacha.network/)

## Problem & Strategy

We cart around a whole set of environment variables to point to the correct environment (staging/prod/pr, hot/Forge, etc.):

* `STORACHA_SERVICE_URL`
* `STORACHA_SERVICE_DID`
* `STORACHA_RECEIPTS_URL`
* `INDEXING_SERVICE_URL`
* `INDEXING_SERVICE_DID`

(These can have slightly names in different codebases.) If these don't all point to the correct values for an environment, things will not go well. That makes pointing to a different service annoying. That affects managing environments, networks, and soon development-local instances of services.

These values must necessarily go together. Rather than set all of them separately, and potentially incorrectly, we can set a single value as an entry point and discover the rest of the service from there. DID documents offer us a great way to do exactly that. Thus, I am proposing that we list the required endpoints and services in the root service's DID document, and discover them from there.

## Background

All Decentralized Identifiers, as defined in [the DID spec](https://www.w3.org/TR/did-1.0/), have a certain set of features. A DID has a *method* which defines a lot how those features are implemented, but all DIDs have the same interface.

Most relevant for this proposal, any DID can be [*resolved* to a DID document](https://www.w3.org/TR/did-1.0/#did-resolution), using the "Read" operation defined by the DID's method. Where the DID itself is an identifier which unambiguously refers to a subject, the DID document describes that subject.

Our subject is the Storacha network. Its description can tell us everything we need to know to properly communicate with it.

Our services are identified with `did:web:` DIDs. Under [that specification](https://w3c-ccg.github.io/did-method-web/), a `did:web:` is resolved by transforming it into a URL and making a GET request. The result should be the DID document. The Forge upload service's DID is `did:web:up.forge.storacha.network`. `did:web:`s can include path elements, but this one doesn't; for such a DID, the URL to fetch is (in this case) `https://up.forge.storacha.network/.well-known/did.json`. We currently serve a DID document there, which looks like:

```json
{
  "@context": [
    "https://w3id.org/did/v1"
  ],
  "id": "did:web:up.forge.storacha.network",
  "verificationMethod": [
    {
      "id": "did:web:up.forge.storacha.network#owner",
      "type": "Ed25519VerificationKey2020",
      "controller": "did:web:up.forge.storacha.network",
      "publicKeyMultibase": "z6MkgSttS3n3R56yGX2Eufvbwc58fphomhAsLoBCZpZJzQbr"
    }
  ],
  "assertionMethod": [
    "did:web:up.forge.storacha.network#owner"
  ],
  "authentication": [
    "did:web:up.forge.storacha.network#owner"
  ]
}
```

This document tells us that the key `z6MkgSttS3n3R56yGX2Eufvbwc58fphomhAsLoBCZpZJzQbr` can be used to verify something signed by `did:web:up.forge.storacha.network`, to assert claims, or for `did:web:up.forge.storacha.network` to authenticate itself with another service.

Even a `did:key:` DID has a DID document, but rather than looking up the document, it's structed programmatically from the information in the DID itself, as [described in the spec](https://w3c-ccg.github.io/did-key-spec/#create). This means that `did:key:` DIDs are immutable: nothing in their documents can be changed. Notably, there is no way to rotate the keys used by a `did:key:` identity (which is really all that the DID document specifies). If the key is compromised, the subject can no longer safely use the identity at all, and must rotate *that*. Conversely, a `did:web:` can rotate its key by simply serving different content as the DID document, and its subject will be able to keep using the same identity.

### Proposal

Along with the properties shown above, [a DID document can also describe services which belong to it](https://www.w3.org/TR/did-1.0/#services). I propose we extend the main service's DID document to include the following:

Would add:

```json
{
  "@context": [
    "https://w3id.org/did/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1",
    "https://storacha.network/ucanto/v1",
    "https://storacha.network/storacha/v1",
  ],
  // ...
  "service": [
    {
      "id": "did:web:up.forge.storacha.network#ucanto-ucan-0.9",
      "type": "Ucanto",
      "serviceEndpoint": "https://up.forge.storacha.network/",
      "receiptsEndpoint": "https://up.forge.storacha.network/receipts/"
    },
    {
      "id": "did:web:up.forge.storacha.network#indexing-service",
      "type": "IndexingService",
      "serviceEndpoint": "did:web:indexer.forge.storacha.network"
    }
  ]
}
```

I've invented the `https://storacha.network/ucanto/v1` and `https://storacha.network/storacha/v1` prefixes here and the names of the types; they may need more thought. The point, though, is that the service types are custom, and that that's perfectly valid. The `IndexingService` here points to another DID, and that can be resolved as well. It would also gain a `service` key with a `Ucanto` service.

A service or client which needs these DIDs and URLs would look these up at startup. A client could potentially cache them if necessary, but since such a client will communicating with the Ucanto services anyway, it may not be an issue to look it up every time. A service can easily do the lookup on boot.