# Big Data from the Gateway

## Authors

- [Alan Shaw](https://github.com/alanshaw), [Storacha Network](https://storacha.network/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Goal

Enable serving big UnixFS encoded files from the gateway.

## Background

The IPFS Gateway `w3s.link` (referred to as "the gateway" subsequently) is currently run by the Storacha Network team. It receives requests for a CID (plus an optional path), assembles UnixFS encoded DAGs into files and sends the resulting bytes back to the client. It has other functionalities but these are not relevant to this RFC.

The gateway is unique in that it does not run a full IPFS node (e.g. Kubo) and does not operate over bitswap. It discovers the location (a URL and other metadata) of blocks via content claims and reads blocks by making HTTP GET requests using byte range headers to extract the exact bytes required.

The gateway is deployed as a Cloudflare worker and performs well for the vast majority of requests. However it has problems serving files that are more than a few hundred megabytes in size.

There are a number of use cases for which this is a requirement. Video is one example of files with size that often exceeds this restriction. OS and docker images, application binaries. Gaming is another big market that requires larger files for assets.

## Problem

Workers are resource constrained. Each invocation operates in 128MiB of RAM and is allocated 30s of CPU time. There is also a maximum number of sub-requests that can be performed.

For files more than a few hundred megabytes a worker consumes all allocated CPU time and the request is terminated. Empirical evidence has shown there is not an exact size that can be cited. A few different factors will be affecting this but it seems that whatever is accounting for worker CPU time is not exact. Chunk size (max size of leaf blocks) within the DAG is another significant factor.

It has been observed that the gateway can only transfer files up to ~500MiB.

Worker termination due to exceeded CPU resources manifests as a truncated response and there is no opportunity for the worker to send an error message in HTTP trailers (for example).

## Proposal

To support larger file transfers we can make a small architecture change and add a worker in front of the current setup. This worker has the following responsibilities:

1. Resolve the CID+path.
2. Decode and inspect the root block to ascertain the size of the file.
3. Make byte ranged sub-requests to the existing infrastructure to extract data required.
4. Combine sub-request streams into a single response stream.

Note: The existing infrastructure MUST be augmented to allow byte range requests to UnixFS files as this is not currently supported.

The thesis is that the worker orchestrating the file transfer will use significantly less CPU time than workers assembling the DAG and hence should be able to transfer much larger files.

## Trade offs

Introducing another worker increases the number of invocations ($$$) per request. We can mitigate impact by making the worker handling the client request to only delegate to sub-workers if the file is found to be big.

When splitting the read into multiple byte range requests, each worker will need to resolve the path and read blocks from the root to the section of the DAG they are interested in ($$$). There will be some overlap here. Including leaf blocks when the byte range does not fall on a block boundary (very likely).

