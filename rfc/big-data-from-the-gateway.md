# Big Data from the Gateway

## Authors

- [Alan Shaw](https://github.com/alanshaw), [Storacha Network](https://storacha.network/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Goal

Enable serving big UnixFS encoded files from the gateway.

## Background

The IPFS Gateway `w3s.link` (referred to as "the gateway" subsequently) is currently run by the Storacha Network team. It receives requests for a CID (plus an optional path), assembles UnixFS encoded DAGs into files and sends the resulting bytes back to the client. It also has other functionalities but these are not relevant to this RFC.

The gateway is unique in that it does not run a full IPFS node (e.g. Kubo) and does not operate over bitswap. It discovers the location (a URL and other metadata) of blocks via content claims and reads blocks by making HTTP GET requests using byte range headers to extract the exact bytes required.

