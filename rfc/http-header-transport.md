# Ucanto transport in HTTP headers

This RFC is an extension of [resourceful UCAN](https://github.com/storacha/RFC/blob/rfc/resourceful-ucan/rfc/resourceful-ucan.md), attempting to further formalize the protocol.

Ideally we'd be able to authorize ourselves to fetch content by simply issuing a HTTP GET request. This is preferable from a caching perspective, but also because it fits with existing content claims (location commitments) that are currenty retrieved as such.

This RFC attempts to define a Ucanto style transport that allows invocations and receipts to be communicated in HTTP headers, leaving the HTTP request/response body usable for alternative purposes. Certain aspects are inspired by the [UCAN as Bearer Token Specification](https://github.com/ucan-wg/ucan-http-bearer-token).

The following HTTP header MUST be included when issuing a UCAN invocation via a HTTP GET _request_:

```yaml
X-Agent-Message: <agent-message-archive>
```

In the header, `<agent-message-archive>` is a CAR file containing an [Agent Message](https://github.com/storacha/go-ucanto/blob/06a2c2d09f708014bda62aba27bb1a146ebd29eb/core/message/datamodel/agentmessage.ipldsch#L1-L8) block, as well an invocation block, optional proof block(s) and optional receipt block(s). The CAR file bytes MUST be gzipped and multibase encoded (base64 is RECOMMENDED).

The Agent Message in _requests_ MUST contain a single invocation. Agent Messages with multiple invocations MUST fail since the response body can only be used by one invocation at a time.

The _response_ headers MUST include an `X-Agent-Message` header, which is an agent message archive that typically contains receipt(s) for the executed invocation tasks. They MUST also include the standard [HTTP `Vary` header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Vary) that includes `X-Agent-Message`.

When sending UCAN invocations via HTTP headers it is important to ensure the total header size does not exceed 8KB, in order to adhere to limits imposed by popular HTTP server software.

It is RECOMMENDED that the `X-Agent-Message` _value_ does not exceed 4KB in size.

To save space and bandwidth an agent MAY omit proofs from the invocation. Especially if making multiple requests to the service using the same proof(s).

The response headers SHOULD include a `X-UCAN-Cache-Expiry` header, set to the cache expiry time in [Unix time](https://en.wikipedia.org/wiki/Unix_time) for proofs referenced by the invocation.

If proofs are omitted in a request and are not present in the server cache, the service MUST respond with a [HTTP 510 (Not Extended)](https://www.rfc-editor.org/rfc/rfc2774#section-7) response. The response body MUST be a [DAG-JSON](https://ipld.io/docs/codecs/known/dag-json/) encoded error object that lists the missing proofs required in order to execute the invocation. It MUST comply to the following schema:

```ipldsch
type MissingProofs struct {
  name    optional String  # Typically "MissingProofs"
  message optional String  # Instructions to resubmit the invocation
  proofs  [UCANLink]       # CIDs of the proofs that were missing
}

type UCANLink = Link  # Link to a UCAN delegation
```

e.g.

```json
{
  "name": "MissingProofs",
  "message": "proofs were missing, resubmit the invocation with the requested proofs",
  "proofs": [
    { "/": "bafyreibd7iyy74awztb3chw73f6yenasubabghjb7jjgu3wjfxrysm7qv4" }
  ]
}
```

The HTTP `Content-Type` header SHOULD be set to `application/json`. Additionally a `X-UCAN-Cache-Expiry` header SHOULD be set to allow the request to be repeated with required proofs, whilst omitting proofs that were already sent.

Note: A repeat invocation MAY omit the original invocation block since it SHOULD be cached by the server.

## Intened usage

As in the [resourceful UCAN RFC](https://github.com/storacha/RFC/blob/rfc/resourceful-ucan/rfc/resourceful-ucan.md) the idea is to use this proposal to allow us to send a `space/content/retrieve` invocation in the HTTP headers of a HTTP GET request. The response body is subsequently used to return the requested bytes in a single round trip (for the majority of requests), while allowing egress to be accounted for via signed invocations/receipts.
