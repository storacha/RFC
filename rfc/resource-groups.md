## Abstract

In w3 family of protocols we often find ourselves contemplating whether to introduce a new [did:web] identifiers like `did:web:web3.storage:free` or introduce additional field like `group` / `tag` / `scope`. In vast majority of use cases additional field is undesired, however overhead of DID provisioning outweights benefits of avoiding extra identifiers.

## BIP32 style derivation

We could use [BIP32] style key deriviation reducing key management overhead, however that does not solve our problem, in fact we do not necessarily need a separate private key, we do want separate indentifier for the same key.

## DID Parameter

We could (ab)use [DID query parameters] to capture additional context e.g `did:web:web3.storage?free`

- 💔 However, DID spec discourages this as it may lead to future incompatibilities
- ❤️‍🩹 ucanto currently would not know how to resolve such DID, but should not be too hard to fix this.
- 💚 This could be a generic solution for above use cases.

## Deriving ownership from hostname

We could address resources (`with` field) using HTTP(S) URLs and derive ownership from hostname. E.g. given a resource `https://web3.storage?plan=free` we can derive resource belongs to `did:web:web3.storage`.

This approach also allows giving access to user namespace to specific keypair

```json
{
  "iss": "did:web:web3.storage",
  "aud": "did:key:zAlice",
  "att": [{
     "with": "https://alice@web.mail@web3.storage"
     "can": "*"
  }]
}
```

[did:web]:https://w3c-ccg.github.io/did-method-web
[BIP32]:https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
[DID query parameters]:https://www.w3.org/TR/did-core/#did-parameters