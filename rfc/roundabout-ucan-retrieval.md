# UCAN Authorized Retrieval via Roundabout

Roundabout acts as a redirect service mapping piece CIDs to their actual location URL(s). It does this by looking up their location on the Storacha Network and then sending a HTTP redirect response.

## Problem

In the Forge network, retrievals are UCAN authorized, necessitating a HTTP header be added to the request that contains a UCAN authorization permitting retrieval.

Roundabout currently just redirects to the found URL. There is no provision for UCAN auth.

## Proposal

The Spade API provides agents (clients) a [frc58 "manifest"](https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0058.md) from which they can build an aggregate piece from.

It contains the aggregate piece CID, a list of _segment_ piece CIDs, and for each segment, a URL (multiple supported, in practice only 1) the data can be retrieved from. It has the structure:

```go
type ResponsePieceManifestFR58 struct {
	AggPCidV2 string    `json:"frc58_aggregate"`
	Segments  []Segment `json:"piece_list"`
}

type Segment struct {
	PCidV2  string   `json:"pcid_v2"`
	Sources []string `json:"sources"` // URL(s) the piece may be fetched from
}
```

We will augment `Segment` sources with header(s) that should be added to HTTP requests:

```go
type Segment struct {
	PCidV2  string   `json:"pcid_v2"`
	Sources []Source `json:"sources"` // URL(s) the piece may be fetched from
}

type Source struct {
  URL string          `json:"url"`
  Headers http.Header `json:"headers,omitempty"`
}
```

The Spade agent (client) would then add the provided headers to the request to fetch data.

At the point the manifest is returned the SP is authenticated via the `FIL-SPID-v0` auth scheme used by Spade. Spade will add a UCAN authorization header (e.g. `X-Agent-Message`) for each segment - a `blob/retrieve` invocation for the specific blob to be retrieved.

It will be necessary for Spade to do the mapping that roundabout currently does, as it needs the blob digest in order to create the `blob/retrieve` invocation. _This process will allow roundabout to be skipped entirely._ as determining the blob digest will also yield the location URL.

The advantage here is that the Spade agent does not need to know how to speak UCAN at all i.e. there is no additional auth needed outside of the existing Spade `FIL-SPID-V0` auth.

## Other options considered

### Signed retrieval URLs

UCAN authorize a signed URL for retrieval. Can include expiration in signed params.

1. Roundabout invokes `access/grant`.
2. Storage node delegates `blob/retrieve/url/sign` to roundabout.

This is cached by roundabout.

When a request comes in, roundabout invokes `blob/retrieve/url/sign` on the storage node, which returns a time restricted signed URL for accessing a blob. This URL is used as the redirect.

The advantage is that Spade does not need to know about UCAN at all.

...but how do we know who is allowed to fetch data?

### Roundabout response includes invocation

How to do? Response is a redirect which is just a URL.

Can we put auth in query string? Probably no.

...but how do we know who is allowed to fetch data?

### Spade agent knows how to talk UCAN

Each Spade agent has a DID.

...but how do we know who is allowed to fetch data?
