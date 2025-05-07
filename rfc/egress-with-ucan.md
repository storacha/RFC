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

## New command *(ability)*: `/space/content/serve` *(`space/content/serve`)*

We add a new command *(UCAN 0.9: ability)* to serve content, named `/space/content/serve` *(`space/content/serve`)*. This command is used to validate the authority to serve content. Notably, it will not (yet) actually *serve* the content, though it may in the future. That is, a successful response will not contain the means to get the content. The Invoker of this command will (for now, at least) always be an entity which *can* get any content in the network (such as the Gateway), but would like to determine whether it *should*.

In the future, this could be invoked from external clients to actually get content, in essence porting the [Gateway spec](https://specs.ipfs.tech/http-gateways/path-gateway/) to UCAN.

### Invocation

```ts
type ServeContent = {
  /**
   * "Serve content owned by the subject Space."
   * 
   * A Principal who may `/space/content/serve` is permitted to serve any
   * content owned by the Space, in the manner of an [IPFS Gateway]. The
   * content may be a Blob stored by a Storage Node, or indexed content stored
   * within such Blobs (ie, Shards).
   * 
   * Note that the args do not currently specify *what* content should be
   * served. Invoking this command does not currently *serve* the content in
   * any way, but merely validates the authority to do so. Currently, the
   * entirety of a Space must use the same authorization, thus the content does
   * not need to be identified. In the future, this command may refer directly
   * to a piece of content by CID.
   * 
   * [IPFS Gateway]: https://specs.ipfs.tech/http-gateways/path-gateway/
   */
  cmd: "/space/content/serve" // (0.9: "space/content/serve")

  /**
   * The Space which contains the content. This Space will be charged egress
   * fees if content is actually retrieved by way of this invocation.
   */
  sub: SpaceDID

  args: {
    /** The authorization token, if any, used for this request. */
    token?: string | null
  }
}
```

### Result

```ts
type ServeContentResult = Result<{}, ServeContentError>

type ServeContentError = {
  message: string
}
```

## Making content retrievable with a token

The Gateway itself will be authorized to `/space/content/serve` by delegation. This signals that the Gateway may serve the data. Typically, this delegation is performed by an Agent within a Storacha Client, who has received the authority to `/space/content/serve` from the Space by way of logging into an Account—thus, the delegation chain flows Space → Account → Agent → Gateway.

```json5
// UCAN 1.0
{
  "iss": "did:key:zAgent",
  "aud": "did:web:w3s.link",
  "sub": "did:key:zSpace",
  "cmd": "/space/content/serve",
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
  "aud": "did:web:w3s.link",
  "att": [
    {
      "with": "did:key:zSpace",
      "can": "space/content/serve",
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

## Making content retrievable *without* a token

The customer may also authorize the Gateway to serve requests with no token. Separately, the Gateway will treat these requests differently: it will rate-limit them and not charge for egress. However, that has no bearing on the UCAN semantics of the delegation.

```json5
// UCAN 1.0
{
  "iss": "did:key:zAgent",
  "aud": "did:web:w3s.link",
  "sub": "did:key:zSpace",
  "cmd": "/space/content/serve",
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
  "aud": "did:web:w3s.link",
  "att": [
    {
      "with": "did:key:zSpace",
      "can": "space/content/serve",
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

The UCAN 1.0 version of these delegations SHOULD specify an explicit `null` policy for the token, rather than include no policy: no policy would match both a `null`/missing token *and* any token, which almost certainly not intended.

The Gateway MAY refuse to serve a request on the authority of an unchecked `token` field. This prevents a malicious actor from generating unexpected egress-incurring requests by including an invented token which will trigger the egress billing. The primary interface for creating these delegations SHOULD provide no means to create such a delegation; given that, the Gateway does not strictly need to check this, as the customer would have to go through some effort to make such a delegation.

The delegation must be available to the Gateway at request-time, so it must have access to it in a store. The authorizing Client should therefore invoke `access/delegate` (UCAN 1.0 equivalent TBD) to store the delegation with Storacha.

## The Gateway

To retrieve a piece of content over HTTP, the HTTP client will connect to the existing [Storacha Gateway](https://w3s.link/) (also known as the web3.storage Gateway). The Gateway offers an HTTP interface which conforms to the [IPFS Subdomain Gateway Spec](https://specs.ipfs.tech/http-gateways/subdomain-gateway/). We extend that spec as follows:

### Request Headers

#### `Authorization` (request header)

Optional. Provides a means of authorizing the content read. Follows the normal [HTTP `Authorization` spec](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization). The authentication scheme must be [`Bearer`](https://datatracker.ietf.org/doc/html/rfc6750). The token is the authorization token the Gateway should use.

```text
Authorization: Bearer abc123def456
```

### Request Query Parameters

#### `auth-token` (request query parameter)

Optional. Provides a means of authorizing the content read. The value is the authorization token the Gateway should use.

```text
https://bafybeidd2gyhagleh47qeg77xqndy2qy3yzn4vkxmk775bg2t5lpuy7pcu.ipfs.w3s.link/?auth-token=abc123def456
```

Requesting clients SHOULD NOT provide both an `Authorization` header and an `auth-token` query parameter. The Gateway MAY use either token if it does, leading to undefined behavior.

### Serving a request

To serve the request, the Gateway will perform the following steps:

1. Look up the Location Commitment(s) for the given root CID from the Indexing Service.
2. If none are found, respond with `404 Not Found`.
3. Note if any Location Commitments were found with no Spaces. (If so, these are from before these changes, and mean we should fall back to the previous behavior later.)
4. Get the set of unique Spaces from those Location Commitments. (There will usually be one Location Commitment, with one Space.)
5. Repeat with each Space in any order, stopping if a successful response is produced:
   1. Look up delegations in the store where the audience is `did:web:w3s.link` and the subject is the Space.
   2. Add them to the collection.
   3. Create an invocation:

      ```json5
      // Shown as UCAN 1.0
      {
        "iss": "did:web:w3s.link",
        "aud": "did:web:w3s.link",
        // The Space
        "sub": "did:key:zSpace",
        "cmd": "/space/content/serve",
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
      * If the invocation fails to authorize,
        * If any Location Commitments were found with no Spaces attached, Perform the existing Gateway file retrieval *with no token*, responding with the file data.
        * Otherwise, respond with `404 Not Found`.
      * If authorization succeeds, execute as:
        1. Perform the existing Gateway file retrieval, passing along the token, responding with the file data.
        2. Asynchronously, inform the Accounting Service of the egress, to be billed to the Space.

After deciding that a given token may retrieve a certain root CID from a certain Space, the Gateway MAY cache the decision by `(root-cid, token)` for some reasonable duration. The Gateway SHOULD NOT cache authorization *failures*, as the customer may be in the process of storing the delegation.

## Hoverboard (Bitswap)

Hoverboard does not handle tokens, but must respect the same authorization semantics as the Gateway does with a non-token request. Because there is no shareable service to perform that auth, Hoverboard will need to reimplement a simpler version of the same logic, only caring about the no-token execution path. This may be improved in the future, but some version of this is required to correctly implement private data storage.

The DID representing Hoverboard is TBD.