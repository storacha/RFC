# Egress, authorized by UCAN

## Authors

- [Petra Jaros](https://github.com/Peeja), [Storacha Network](https://storacha.network/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Goals

* We (Storacha) want to charge customer for data egress when retrieving data through our Gateway.
* We want to provide the same free option we offer today, with a rate-limit.
* We want to allow customers to make their data private: inaccessible without a token and inaccessible through Bitswap.

## Abstract

A proposed means for offering content egress from the Storacha Network which can be selectively authorized by UCAN, and which can track egress to bill appropriate customers. As usual within the Storacha Network, we begin from self-sovereignty: a Space definitionally has the authority to do anything with its content.

To make content available as on a traditional CDN, an Agent acts on authority of a Space to authorize the Gateway to serve that Space's contents to anyone bearing a token, an opaque, unguessable string. The Gateway then responds to requests which contain a token by validating the proof chain, finding the content, charging for egress, and proxying it to the requester.

This process does *not* include making and tracking Location Commitments. A Location Commitment is an attestation by a Storage Node that it holds a particular piece of content on behalf of a particular Space, and that it can provide it. In this proposal, we assume such a system already exists.

## Amend command *(ability)*: Get Blob

We will modify the [Get Blob](https://github.com/storacha/specs/blob/main/w3-blob.md#get-blob) UCAN command *(or in UCAN 0.9, ability)*.

First, we update the original spec to reality. Where the spec names the command `/space/content/get/blob/0/1`, we have implemented it in UCAN 0.9 as `space/blob/get/0/1`. We will continue to use that name, and use that name in this document, as it provides a better authorization hierarchy—specifically, the Space can grant a Principal `space/blob/*`.

Second, we will add an additional argument, `token`. This represents the bearer token when invoked through the Gateway, or something similar. It can be `null` or missing, and will be in existing uses of the command.

Note that the command has a `/0/1` suffix. This appears to be included to mark it as unstable. This proposal does not include removing the suffix, though that will likely make sense to do soon.

```ts
type GetBlob = {
  /** "Retrieve details about a Blob from a Space." */
  cmd: "/space/blob/get/0/1" // (0.9: "space/blob/get/0/1")

  /**
   * The Space which contains the Blob. This Space will be charged egress fees
   * if the Blob data is actually retrieved by way of this invocation.
   */
  sub: SpaceDID

  args: {
    /** The multihash digest of the Blob to look up. */
    digest: Uint8Array

    /** The token, if any, used for this request. */
    token?: string | null
  }
}
```

The receipt type remains [unchanged](https://github.com/storacha/specs/blob/main/w3-blob.md#get-blob-receipt).

### Existing uses

The existing use of this command is to read metadata about the Blob, specifically the size. For instance the Console uses this to display the size of a file Shard in the UI. This use remains unchanged. The `token` should be left missing, or set to `null`.

## Making content retrievable with a token

The Gateway itself will be authorized to `/space/blob/get/0/1` by delegation. This signals that the Gateway may serve the data. Typically, this delegation is performed by an Agent within a Storacha Client, who has received the authority to `/space/blob/get/0/1` from the Space by way of logging into an Account—thus, the delegation chain flows Space → Account → Agent → Gateway.

```json5
// UCAN 1.0
{
  "iss": "did:key:zAgent",
  "aud": "did:web:ipfs.w3s.link",
  "sub": "did:key:zSpace",
  "cmd": "/space/blob/get/0/1",
  "pol": [
    ["==", ".token", "abc123def456"],
  ],
  "nonce": …,
  "exp": …
}
```

```json5
// UCAN 0.9
{
  "iss": "did:key:zAgent",
  "aud": "did:web:ipfs.w3s.link",
  "att": [
    {
      "with": "did:key:zSpace",
      "can": "space/blob/get/0/1",
      "nb": {
        "token": "abc123def456"
      },
    }
  ],
  "nnc": …,
  "exp": …,
  "prf": […]
}
```

Such delegations SHOULD NOT specify a `digest`, as this will be the digest of a Shard Blob, not the file which it contains part or all of. We are not (currently) supporting a use case in which the files in a Space have different access permissions from one another: all permissions should apply to an entire Space.

The UCAN 1.0 version of these delegations SHOULD specify a top-level policy of the form `["==", ".token", <string>]`. This prevents customers from shooting themselves in the foot by getting too fancy with their tokens, which is not a supported use case, or one we see value in. Such use cases may be considered for support in the future.

## Making content retrievable *without* a token

The customer may also authorize the Gateway to serve requests with no token. Separately, the Gateway will treat these requests differently: it will rate-limit them and not charge for egress. However, that has no bearing on the UCAN semantics of the delegation.

```json5
// UCAN 1.0
{
  "iss": "did:key:zAgent",
  "aud": "did:web:ipfs.w3s.link",
  "sub": "did:key:zSpace",
  "cmd": "/space/blob/get/0/1",
  "pol": [
    ["==", ".token", null],
  ],
  "nonce": …,
  "exp": …
}
```

```json5
// UCAN 0.9
{
  "iss": "did:key:zAgent",
  "aud": "did:web:ipfs.w3s.link",
  "att": [
    {
      "with": "did:key:zSpace",
      "can": "space/blob/get/0/1",
      "nb": {
        "token": null
      },
    }
  ],
  "nnc": …,
  "exp": …,
  "prf": […]
}
```

The UCAN 1.0 version of these delegations MUST specify an explicit `null` policy for the token, rather than include no policy. The Gateway MUST refuse to serve a request on the authority of an unchecked `token` field. This prevents a malicious actor from generating unexpected egress-incurring requests by including an invented token which will trigger the egress billing.

The delegation must be available to the Gateway at request-time, so it must have access to it in a store. The authorizing Client should therefore invoke `access/delegate` (UCAN 1.0 equivalent TBD) to store the delegation with Storacha.

## The Gateway

To retrieve a piece of content over HTTP, the HTTP client will connect to the existing Storacha Gateway (also known as the web3.storage Gateway). Later, other entities (either Storage Providers or customers themselves, or entirely separate parties) may implement their own Gateways, which may or may not follow the behaviors here. This specification applies only to the Storacha Gateway itself.

The Gateway will offer an HTTP endpoint. Currently, the Storacha Gateway's endpoint takes the form of `https://<root-cid>.ipfs.w3s.link/<optional-path-and-query-params>`. (Note that any query params are currently ignored by the Gateway, and could only have any effect on the client, such as JavaScript in a served HTML document reading them.) The Gateway will now accept a token as part of the URL, as an `authToken` query param, which may be among any other (ignored) query params. It will also recognize a token provided in the request headers as `Authorization: Bearer <token>`.

To serve the request, the Gateway will perform the following steps:

1. Look up the Location Commitment(s) for the given root CID from the Indexing Service.
   * If not found, fall through to the Gateway's prior retrieval behavior. This is a deprecated branch of behavior to serve data stored before the advent of the Indexing Service, which will launch at the same time as these changes. Only data added after these changes are launched is subject to this authorization scheme.
2. Get the set of unique Spaces from those Location Commitments. (There will usually be one Location Commitment, with one Space.)
3. Repeat with each Space in any order, stopping if a successful response is produced:
   1. Look up delegations in the store where the audience is `did:web:ipfs.w3s.link` and the subject is the Space.
   2. Add them to the collection.
   3. Create an invocation:

      ```json5
      // Shown as UCAN 1.0
      {
        "iss": "did:web:ipfs.w3s.link",
        "aud": "did:web:ipfs.w3s.link",
        // The Space
        "sub": "did:key:zSpace",
        "cmd": "/space/blob/get/0/1",
        "args": {
          // The `authToken` in the HTTP request, or...
          "token": "the-token",
          // null if none was given.
          "token": null
        },
        // Random
        "nonce": { "/": "abc123" },
        "exp": null,
        "prf": [
           // The found delegations
        ]
      }
      ```

   4. Attempt to execute the invocation.
      * If the invocation fails to authorize, respond with 401 Unauthorized.
      * If authorization succeeds, execute as:
        1. Perform the existing Gateway file retrieval, responding with the file data.
        2. Asynchronously, inform the Accounting Service of the egress, to be billed to the Space.

After deciding that a given token may retrieve a certain root CID from a certain Space, the Gateway MAY cache the decision by `(root-cid, token)` for some reasonable duration. The Gateway SHOULD NOT cache authorization *failures*, as the customer may be in the process of storing the delegation.

## Hoverboard (Bitswap)

Hoverboard does not handle tokens, but must respect the same authorization semantics as the Gateway does with a non-token request. Because there is no shareable service to perform that auth, Hoverboard will need to reimplement a simpler version of the same logic, only caring about the no-token execution path. This may be improved in the future, but some version of this is required to correctly implement private data storage.

## Potential future work

We may eventually create an object-store service. This service will sit between the Gateway and the Storage Nodes. It will speak UCAN, while the Storage Nodes will continue to speak only HTTP byte-range requests. This service will be the Executor of these `/space/blob/get/0/1` invocations, rather than the Gateway. This will allow other Principals to use the same mechanisms to retrieve Blobs, including Hoverboard as well as customers storing and retrieving Blobs directly. It would return a means to fetch the actual data in the receipt for the invocation, such as an HTTP URL.

## Open questions

* When auth fails, should we return 401 Unauthorized, or 404 Not Found to mask that the data exists?
* Should `token` be `gatewayToken`? Or `authToken`? Is `token` too broad a term to be in the args of that invocation?
* What if we find a Location Commitment, but it has no Space on it (since [it's optional](https://github.com/storacha/indexing-service/pull/18))?
