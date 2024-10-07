# Egress, authorized by UCAN

## Goals

* We (Storacha) want to charge users for data egress when retrieving data through our Gateway.
* We want to provide the same free option we offer today, with a rate-limit.
* We want to continue to provide access through Bitswap, but make it optional.
* We eventually want to allow Storage Nodes to offer their own direct egress, bypassing our Gateway, and to charge the customer according to an agreement with them.

## Abstract

A proposed means for offering content egress from the Storacha Network which can be selectively authorized by UCAN, and which can track egress to bill appropriate customers. As usual within the Storacha Network, we begin from self-sovereignty: a Space definitionally has the authority to do anything with its content.

To make content available as on a traditional CDN, an Agent acts on authority of a Space to make that Space's contents, or some specific CID in it, available using a bearer token, an opaque, unguessable string. The Gateway then responds to requests which contain a token by validating the proof chain, finding the content, charging for egress, and proxying it to the requester.

This process does *not* include making and tracking Location Commitments. A Location Commitment is an attestation by a Storage Node that it holds a particular piece of content on behalf of a particular Space, and that it can provide it. In this proposal, we assume such a system already exists.

## A new command *(ability)*: `/space/content/retrieve` *(`space/content/retrieve`)*

We will add a new UCAN command *(or in UCAN 0.9, ability)* to our system, called `/space/content/retrieve` *(or in UCAN 0.9, `space/content/retrieve`)*.

> **Command:** `/space/content/retrieve` *(0.9: `space/content/retrieve`)*
> 
> * **Meaning:** To retrieve a piece of content from a Space.
> * **Subject:** The Space from which content will be retrieved.
> * **Arguments *(0.9: `nb`)*:**
>   * `cid`: The CID of the content which will be retrieved.
> * **Receipt:** [TBD, but must provide instructions to access the data (using HTTP?) without further UCAN authorization.]

The Space may then delegate this to another Principal to give them authority to access the Space's content. Typically, this will not be done directly (though it may), but indirectly through an Account and an Agent: the Space will delegate all of its capability to an Account, which will delegate all of *its* capability to an Agent when it logs in. Then the Agent (ie, the logged-in customer) can share access to the content as they see fit.

## A new DID method: `did:bearer`

Sometimes, it's impractical to delegate retrieval to every entity which may want to get the content from a Space. For instance, ordinary users browsing a website cannot reasonably acquire a delegation giving them access to each image on the page. The images must be bound to the Space's billing account, and should be rate-limitable to avoid excessive charges and abuse, but also must be accessible with a simple HTTP GET.

Traditional content delivery networks often use a **token** for analogous purposes. A CDN customer receives a token from the service which they attach to the URLs of their assets. When those URLs are fetched, the CDN uses the token to charge the correct customer and to allow the customer to shut off abusive access. The token is a bearer token: simply knowing and presenting the token is enough to authorize the HTTP request.

The `did:bearer` method provides a similar scheme within a DID, which can then be applied to UCAN. The DID includes a token, and anyone who presents that token is by definition authenticated as that DID. This allows for a simple authorization bridge from an HTTP bearer token scheme to a UCAN delegation chain.

### Format

The format for the `did:bearer` method conforms to the [DID specification](https://www.w3.org/TR/did-core/). It consists of the `did:bearer` prefix, followed by the percent-encoded token. The ABNF for the format is described below, using the ABNF syntax described in [RFC 5234](https://www.rfc-editor.org/rfc/rfc5234). The definition of `idchar` is found in the [DID syntax](https://www.w3.org/TR/did-core/#did-syntax).

```rfc5234
did-bearer-format = "did:bearer:" idchar
```

### Operations

The following section outlines the DID operations for the `did:bearer` method.

#### Create

Creating a `did:bearer` value begins with a token (a string) which it should represent.

1. Percent-encode the token value, as specified in [RFC 3986 Section 2.1](https://www.rfc-editor.org/rfc/rfc3986#section-2.1).
2. Prepend the result with `did:bearer:`.

For example, the token `abc$*)123` would be represented by the DID `did:bearer:abc%24%2a%29123`. Anyone presenting the token `abc$*)123` by any means a service accepts may be considered by that service to be the [DID subject](https://www.w3.org/TR/did-core/#dfn-did-subjects) of `did:bearer:abc%24%2a%29123`.

The DID Document for a `did:bearer` DID is extremely simple. A `did:bearer` DID is associated with no cryptographic keys. It is not capable of signing. All authentication occurs out of band, so it has no verification methods. The DID Document is therefore trivially generated from the DID. An example is given below that expands the DID `did:bearer:abcde12345` into its associated DID Document:

```json
{
  "@context": "https://www.w3.org/ns/did/v1",
  "id": "did:bearer:abcde12345"
}
```

#### Read

Reading a `did:bearer` value is a matter of deterministically expanding the value to a DID Document. This process is described above in [Create](#create).

#### Update

This DID Method does not support updating the DID Document.

#### Deactivate

This DID Method does not support deactivating the DID Document.

## Making content retrievable with a token

A UCAN delegation is used to allow token access to content. Typically, this is performed by an Agent within a Storacha Client, who has the authority to `/space/content/retrieve` from the Space by way of logging into an Account—thus, the delegation chain flows Space → Account → Agent → Token.

```json5
// UCAN 1.0
{
  "iss": "did:key:zAgent",
  "aud": "did:bearer:abcde12345",
  "sub": "did:key:zSpace",
  "cmd": "/space/content/retrieve",
  "pol": [
    ["==", ".cid", "bafy...7pcu"],
  ],
  "nonce": …,
  "exp": …
}
```

```json5
// UCAN 0.9
{
  "iss": "did:key:zAgent",
  "aud": "did:bearer:abcde12345",
  "att": [
    {
      "with": "did:key:zSpace",
      "can": "space/content/retrieve",
      "nb": {
        "cid": "bafy...7pcu"
      },
    }
  ],
  "nnc": …,
  "exp": …,
  "prf": […]
}
```

The delegation must be available to the Executor at invocation-time. Since the Invoker will be using a token and not speaking UCAN, they will not be able to deliver the proof, so the Executor must have access to it in a store. The Client should therefore invoke `access/delegate` (UCAN 1.0 equivalent TBD) to store the delegation with Storacha.

## The Gateway

To retrieve a piece of content over HTTP, the Retriever will connect to a Gateway. At first, there will be exactly one Gateway: the existing Storacha Gateway (also known as the web3.storage Gateway). Later, as Storage Nodes become more sophisticated, they may decide to run their own Gateways, offering their own direct egress to customers.

The Gateway will offer an HTTP endpoint. Currently, the Storacha Gateway's endpoint takes the form of `https://<cid>.ipfs.w3s.link/`. The Gateway will accept a token as part of the URL. To serve the request, the Gateway will perform the following steps:

1. Look up the Location Commitment for the given CID. If not found, respond with 404 Not Found.
2. If there is no token given, serve the request through the standard rate-limiter. (This behavior is likely to change in the future.)
3. If there is a token given (eg, `abcde12345`), build its corresponding DID (eg, `did:bearer:abcde12345`).
4. Look up all delegations with the token DID as audience.
5. Attempt to prove the ability to invoke `/space/content/retrieve` on the Space listed in Location Commitment, with the given CID. If more than one Location Commitment is found, attempt each in turn: a CID may be stored in multiple Spaces, and the token may be able to retrieve the content through one and not another.
6. If no such proof chain is available, respond with 401 Unauthorized. [or 404 Not Found?]
7. If a proof chain is found, execute the invocation on the Executor.
8. Using the information in the receipt, fetch the content and proxy it as the response.

## The Executor

The [Executor](https://github.com/ucan-wg/invocation?tab=readme-ov-file#executor) of the `/space/content/retrieve` invocation will be the same service as the Gateway. [It would also be possible for Storage Nodes to run their own Executors for the central Storacha Gateway to use, but this would require more sophistication from the Storage Nodes without the benefit to them of offering their own egress service.]

To execute the invocation, the Executor will perform the following steps:

1. Validate the proof chain.
2. [Look up the Location Commitment again? :/ Should it become an argument?]
3. [Charge the Space for the egress.]
4. [Produce a suitable receipt.]

## [To Come]

* Rather than serve non-token content rate-limited by default, require a delegation of `/space/content/retrieve` to some DID representing "anyone".
* Bitswap should execute a `/space/content/retrieve` as well to respond to requests. Bitswap should be authorized by delegating to some DID representing Bitswap/Hoverboard.

## Open questions

* What does the `/space/content/retrieve` receipt look like?
* Can you enumerate the contents of a space? Is that in scope here?
* What does a Gateway URLs with a token look like?
* What can be cached? This seems relatively cacheable, but we should be explicit in the design to make sure we're on a suitable path. This process needs to be *fast*, at least once the cache is warm.
