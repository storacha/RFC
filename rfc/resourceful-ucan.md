# Large Resource Downloads With UCAN

## Authors

- [hannahhoward]

## Goals

Define a standard for fetching larger HTTP resources in UCAN invocations

## Problem Statement

Our UCAN invocation library Ucanto does not use GET requests or unique URLs for resources. Instead, it assumes a single POST endpoint for all invocations.  This is slightly more akin to a GraphQL approach, where we step outside HTTP REST principles via a custom RPC API. While this make sense for APIs that are truly RPC, i.e dealing with dynamic data and executing short-lived remote commands, what about the case where we simply want to use a ucan invocation to authorize the download of a (potentially large) static resource?

Here are some reasons we might consider a different approach for this use case:

1. UCanto adds additional complexity of having to extract a resource from a UCAN receipt envelope, in a context where the UCAN itself is likely encoded in a CAR file. That complexity has to be handled by both server and client

2. Authorization layers are frequently implemented as proxys on top of a simpler non-internet facing HTTP API (i.e. a simple disk read for example). Forcing a full CAR->CAR endpoint means either the underlying HTTP API must get more complex or the proxy must do complex response transformation.

3. Designing UCAN invocation APIs that use resourceful URLs reduces cognitive overhead for those building UCAN services -- the less UCANs deviate from REST conventions, the less developers have to change their mindset to write them.

Is there a way to provide a simplified API that offers the security of UCAN invocations without the complexity of UCanto's default HTTP transport, specifically for the use case of authorizing download of static resources?

## Observations

1. Receipts are essentially optional concepts in UCAN. The invocation spec is actually not a "request-response" protocol, especially in UCAN 1.0. In 1.0 a receipt is just an invocation for a 'ucan/assert' capability.

2. The transport mechanism for HTTP in UCanto is NOT actually at all hardcoded, nor is the encoding format. The framework is specifically designed to support other transport interfaces. With that said, Ucanto's code currently expects responses to be returned as transactions -- a receipt and effects, with no facility for a streaming response body.

## Possible Approaches

### HTTP POST with invocation in request body and resource in response body

The simplest and most flexible proposal is to use traditional resource URLs, but with an POST instead of a GET in order to include an invocation in the request body.

Request: An HTTP POST with a complete UCAN invocation in the request body, encoded either as a single binary DAG-CBOR block or as CAR file of multiple blocks

Response: Resource in response body, receipt/transaction encoded an X-UCAN-Response header, gzipped and written as a UTF-8 string.

#### Tradeoffs

- 💚 Response body unaltered
- 💚 Can drop the request body and convert to GET request to internal API when operating as a proxy
- 💔 Still sizable deviation from REST -- POST for the purpose of fetching a resource is very unusual. While POST is generally the escape hatch HTTP verb for when you just want to do whatever, it's still a fairly unexpected use case
- 💔 UCAN in a response header limits the acceptable size of UCAN receipt

### HTTP GET with invocation in the authorization header and resource in the response body

In this proposal, we use a traditional resource URL along with the expected  fetch verb, HTTP GET. We include the invocation in the Authorization header of the request.

Request: An HTTP GET with a complete UCAN invocation encoded as a single DAG-CBOR block, then GZipped, then serialized to a base64 string and passed as a bearer token in the Authorization header.

Response: Resource in response body, receipt/transaction encoded an X-UCAN-Response header, gzipped and written as a UTF-8 string.

#### Tradeoffs

- 💚 Response body unaltered
- 💚 Can simply pass on the request as is to the internal API (possibly dropping the request header)
- 💚 Close match to expected HTTP REST API behavior
- 💔 Base64 UCAN in Authorization header may severely limit the size of allowed UCAN invocation
- 💔 UCAN in a response header limits the acceptable size of UCAN receipt

### HTTP GET with abbreviated invocation in the authorization header, resource in the response body, seperate endpoint to store delegation chain

This proposal is a variation on the above HTTP GET proposal where we provide a mechanism for avoiding issues with the length of the invocation by storing the delegation chain (i.e. prf field) as a set of CID, rather than a full delegation chain. Seperately, we provide an endpoint for storing the delegation chain for the agent we indend to use for invocations (access/delegate protocol should work unaltered). 

Request: An HTTP GET with a UCAN invocation where the delegation chain is only CIDs encoded as a single DAG-CBOR block, then GZipped, then serialized to a base64 string and passed as a bearer token in the Authorization header.

Response: Resource in response body, receipt/transaction encoded an X-UCAN-Response header, gzipped and written as a UTF-8 string.

#### Tradeoffs

- 💚 Response body unaltered
- 💚 Can simply pass on the request as is to the internal API (possibly dropping the request header)
- 💚 Close match to expected HTTP REST API behavior
- 💚 Base64 UCAN in request header unlikely to be too large absent delegation chain
- 💔 UCAN in a response header limits the acceptable size of UCAN receipt

- 💔 Extra HTTP request to store delegation chain
- 💔 Server has to store information on behalf of client


## Analyzing UCAN Size Limits

In considering the second and third proposed approach, it's worth asking why UCAN invocations are not traditionally passed as an Authorization header to a GET request. In fact, in the original draft of UCAN, delegations were proposed as an extension to JWTs. This suggests they were designed to be put in an Authorization header.

Unfortunately, most HTTP clients and servers will not process by default HTTP header blocks where the total size is greater than four kilobytes. While a shallow delegation chain can fit within this contraint, longer delegations can easily go over. This is even more pronounced in the Authorization header, here a Bearer token is expected to be encoded in base64 as opposed to simply stuffing binary data into a utf-8 string. 

In practice, in Storacha we often generate longer delegation chains, particularly when an agent claims access to a space generated by another agent through a common Storacha account. The table below covers byte counts for a delegation of the `space/content/*` capability to a did:key agent for a space that the delegating agent claimed via access to a Storacha account (creating a longer delegation chain).

| Encoding steps | Size | Diff to raw DAG-CBOR |
| ---- | ---- | --- |
| Raw DAG-CBOR | 1960 | 0 |
| DAG-CBOR base64 | 2617 | +657 |
| DAG-CBOR gzipped | 1177 | -783 |
| DAG-CBOR gzipped base64 | 1573 | -387 |

Note these are delegation chains, and are missing the one additional wrapper+signature of a UCAN invocation.

Nevertheless, there are some interesting learnings here:
1. DAG-CBOR UCAN's appear to compress well with gzip (which makes sense given keys are often repeated)
2. Even for these longer authorization chains, UCANs may fit in request headers particularly when compressed with GZIP

Either way, we might lean towards an HTTP GET approach that orients towards simply putting full UCANs in request headers, but falls back to storing the delegation chain in rare cases we can't encode the UCAN in a header.

### Alternative for HTTP GET approaches: use X-UCAN-Request custom header

One additional possible improvement for GET approaches is to use an `X-UCAN-Request` custom header instead of the Authorization header for UCAN invocations.

There are a couple pros and cons to consider here:
- 💚 Encoding gzipped UCAN DAG-CBOR data in raw UTF-8 strings instead of base64 allows greater compression
- 💚 Perhaps a UCAN invocation isn't simply an "Authorization" anyway, and belongs in a custom header
- 💔 Sending binary data in HTTP headers as UTF-8 strings is fairly non-standard
- 💔 Custom headers require attention to CORS issues and may get dropped more often by proxy servers (this applies to X-UCAN-Response as well)

## Storage Node Retrieval Protocol Proposal

For Storacha warm storage nodes, use HTTP GET requests with UCAN invocations of `space/content/retrieve` encoded to DAG-CBOR, gzipped, and passed as a base64 encoded Bearer token in the HTTP Authorization header. Particularly for warm storage, the client agent will more typically be the original creator of a space, making for a shorter delegation chain. Measure the invocation size GZipped ahead of time, and if larger than three kilobytes, store the delegation chain using access/delegate with the warm storage node, then for subsequent requests create an invocation with the delegation chain listed as a CID. (this will require adding access/delegate to the supported protocols on warm storage nodes, with some sort of hard coded per-space/agent max storage size)

Using HTTP GET and resourceful URLs will enable us to implement `space/content/retrieve` with no changes to existing location commitment logic

`space/content/retrieve` will be specified in more detail in a PR to https://github.com/storacha/specs