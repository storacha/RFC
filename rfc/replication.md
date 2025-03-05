# RFC: Replication

## Authors

- [Alan Shaw](https://github.com/alanshaw), [Storacha Network](https://storacha.network/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Introduction

Replication ensures that if a storage node loses a blob, it remains available in the network. This RFC proposes an extension to the blob protocol that allows nodes within the network to replicate data between each other _after_ a storage node has received an initial upload.

Out of scope: This RFC does not propose any solution for repairing lost data from any node.

## Proposal

A `space/blob/replicate` invocation instructs the upload service to replicate a blob to the specified number of storage nodes. A replication request is only valid to be issued _after_ the client has received a `blob/accept` receipt, indicating the target storage node has successfully received the blob.

The caveats for `space/blob/replicate` MUST include a `replicas` parameter, an unsigned integer indicating the number of copies the network should ensure are stored _in addition_ to the original.

It is RECOMMENDED that the upload service mandate a minimum _and_ a maximum value. The minimum and maximum MAY be the same.

e.g. `replicas: 2` will ensure 3 copies of the data exist in the network.

```json5
{
  "iss": "did:key:zAlice",
  "aud": "did:web:upload.service.example.com",
  "att": [
    {
      "with": "did:key:zAliceSpace",
      "can": "space/blob/replicate",
      "nb": {
        /** The blob that MUST be replicated. */
        "blob": {
          "digest": { "bytes": "..." },
          "size": 1234
        },
        /** Number of replicas to ensure. */
        "replicas": 2,
        /** A location commitment indicating where the blob MUST be fetched from. */
        "location": { "/": "bafy..locationCommitment" }
      }
    }
  ],
  "prf": [],
  "sig": "..."
}
```

It is RECOMMENDED that the location commitment be included in the invocation.

The receipt for `space/blob/replicate` includes effects (async tasks) for `blob/replica/accept`. Successful completion of the `blob/replica/accept` task indicates the replication target has stored the blob. The number of `blob/replica/accept` tasks corresponds directly to number of replicas requested.

A `blob/replica/accept` task takes the following form:

```json5
{
  "iss": "did:web:upload.service.example.com",
  "aud": "did:key:zStorageNode",
  "att": [
    {
      "with": "did:key:zStorageNode",
      "can": "blob/replica/accept",
      "nb": {
        /** The blob that was replicated. */
        "blob": {
          "digest": { "bytes": "..." },
          "size": 1234
        },
        /** DID of the space the blob has been allocated to. */
        "space": { "bytes": "..." },
      }
    }
  ],
  "prf": [],
  "sig": "..."
}
```

Each `blob/replica/accept` task MUST target a _different_ storage node and they MUST NOT target the original upload target.

The upload service MUST select storage node(s) and allocate replication space when the `space/blob/replicate` invocation is received. The upload service allocates replication space by issuing a `blob/replica/allocate` invocation:

```json5
{
  "iss": "did:web:upload.service.example.com",
  "aud": "did:key:zStorageNode",
  "att": [
    {
      "with": "did:key:zStorageNode",
      "can": "blob/replica/allocate",
      "nb": {
        /** The blob that was must be replicated. */
        "blob": {
          "digest": { "bytes": "..." },
          "size": 1234
        },
        /** DID of the space the blob has been allocated to. */
        "space": { "bytes": "..." },
        /** A location commitment indicating where the blob MUST be fetched from. */
        "location": { "/": "bafy..locationCommitment" },
        /** The `space/blob/replicate` invocation that caused this allocation. */
        "cause": { "/": "bafy..replicate" }
      }
    }
  ],
  "prf": [],
  "sig": "..."
}
```

It is RECOMMENDED that the original replicate invocation (the cause) and the location commitment be included in the invocation.

A storage node that receives a `blob/replica/allocate` should fetch and store the blob from the specified location. A receipt should be issued for the `blob/replica/accept` task.

Client can poll for `blob/replica/accept` receipt.
