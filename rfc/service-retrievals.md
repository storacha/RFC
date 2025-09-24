# Service Retrievals

There are a number of occasions where internal services within the Storacha Network need to retrieve data from Storage Nodes:

1. The _Upload Service_ retrieves indexes as part of an `space/index/add` invocation, so that it can publish all the hashes to IPNI.
2. The _Indexing Service_ fetches blob indexes when an `assert/index` claim is published to it, as well as part of the normal flow when querying for hashes.
3. The _Filecoin Service_ fetches all data submitted to it in order to validate the provided Piece CID matches the data.
4. Other _Storage Nodes_ fetch blobs when instructed to replicate data.

This RFC lays out 3 options for authorizing retrieval of this content.

## Pre-Delegated Retrieval via `space/content/retrieve`

Pre-delegated retrieval is the idea that delegations can be created by a client and stashed with various services, for use when required. We use this pattern in our gateway, where it is oftentimes not feasible for clients to generate a delegation per request.

* 🔴 Need to permanently store delegations for all content (or at minimum 1 delegation per space).
* 🔴 Engineering work to update client libraries to authorize services when space or content is created and also to allow delegations to be stashed with each service.
* 🟢 Would be backwards compatible - no delegation = try regular HTTP retrieval.
* 🟢 Authorizer pays for the egress.

## On-Demand Delegated Retrieval via `space/content/retrieve`

On-demand delegated retrieval is the process of generating a delegation and attaching it to an invocation at the point at which it is needed.

* 🔴 Engineering work required to extract delegation and use it in a UCAN retrieval request for each invocation where it is needed.
* 🟠 Need to delegate an open ended capability to the indexer since the CID of an index it may need to retrieve is not known beforehand.
* 🟢 Would be backwards compatible - no delegation = try regular HTTP retrieval.
* 🟢 Authorizer pays for the egress.

## Service Retrieval via `blob/retrieve` (name TBC)

Storage Nodes delegate a retrieval capability to the upload service, indexing service and filecoin service giving them access to retrieve blobs as required. This can be done while onboarding. Note, this is a variation of Pre-Delegated Retrieval (as above).

* 🔴 Centralized and privileged access to data that is not directly authorized by the data owner.
* 🔴 Need to permanently store another delegation per storage node for each service.
* 🔴 ??? pays for the egress.
* 🟠 Engineering work to update piri and delegator for new delegation, also new capability definition and handler implementation.
* 🟢 Would be backwards compatible - no delegation = try regular HTTP retrieval.
