# RFC: Mutability for Forge (Guppy/Piri)

**Status: Draft - Pending Alignment**

## Authors

- [Felipe Forbeck](https://github.com/fforbeck), [Storacha Network](https://storacha.network/)

## Editors
- [Alan Shaw](https://github.com/alanshaw), [Storacha Network](https://storacha.network/)
- [Hannah Howard](https://github.com/hannahhoward), [Storacha Network](https://storacha.network/)
- [Alex Kinstler](https://github.com/prodalex), [Storacha Network](https://storacha.network/)


## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Introduction

This RFC proposes the implementation of mutability features for Forge enterprise customers using the Guppy client. These features enable:

1. **Mutable References**: Stable pointers that update to the latest version of uploaded content
2. **Content Catalog**: Track uploaded content (path to CID mappings) across clients

**Related:** [forge-encryption.md](./forge-encryption.md) - Encryption key rotation depends on mutability to track metadata CID changes.

## Motivation

Enterprise Forge customers need:

- **Mutable references**: Backup workflows need stable names that update to point to the latest backup version
- **Cross-client sync**: Multiple Guppy instances should be able to resolve the same reference
- **On-network state**: Track what has been uploaded without relying on local-only databases

## Out of Scope

- **All-file-paths indexing**: Storing every individual file path in Pail (e.g., `/backups/mydir/file1.txt`, `/backups/mydir/subdir/file2.txt`). This RFC only tracks the source root path to root CID mapping. Internal file structure remains in UnixFS.

## Approach: UCN + Pail

Forge will use **UCN** + **Pail** for mutable content tracking:

- **UCN**: Lightweight wrapper around `clock/head` and `clock/advance` for publishing/resolving names
- **Pail**: Sharded Merkle trie for `path -> CID` mappings (scales to millions of files)
- **Clock service**: Already exists at `clock.web3.storage`

### Why Pail?

Forge directories can contain thousands to millions of files. Pail's sharded structure means only changed shards are uploaded when a file is added or updated.

**Note:** Guppy continues to upload content as UnixFS (for traditional retrieval patterns). Pail stores the mapping from source path to the UnixFS root CID, enabling path-based lookups without changing the upload format.

### Architecture

```mermaid
graph LR
    subgraph Guppy
        A[Upload file] --> B[Update Pail]
        B --> C[Publish via UCN]
    end
    
    C -->|clock/advance| D[Clock Service]
    B -->|store blocks| E[Storage Network]
    
    subgraph Resolve
        F[name.Resolve] -->|clock/head| D
        F --> G[crdt.Root]
        G --> H[pail.Get]
    end
```

### What to Build

| Component | Status |
|-----------|--------|
| UCN wrapper (`clock/head`, `clock/advance`) | Needs Go impl (~few hundred lines) |
| Pail integration | Library exists: `github.com/storacha/go-pail` |
| Guppy commands | Update `upload`, `ls`, `retrieve` |

## Integration with Guppy Flows

### Current Guppy Upload Pipeline

Guppy's `ExecuteUpload` runs a pipeline of workers:

1. **Scan Worker** - Walks filesystem, creates FSEntry records
2. **DAG Scan Worker** - Creates DAG nodes from files, sets rootCID
3. **Sharding Worker** - Packs nodes into CAR shards
4. **Indexing Worker** - Creates indexes for shards
5. **Shard Upload Worker** - Uploads shards via `space/blob/add`
6. **Index Upload Worker** - Uploads indexes via `space/blob/add`
7. **Post-Process Workers** - Finalizes shards/indexes, calls `upload/add`

Returns: `rootCID` (the content root)

### Upload with Mutability (Proposed)

```mermaid
sequenceDiagram
    participant User
    participant Guppy
    participant Workers
    participant Storacha
    participant Pail
    participant UCN
    
    User->>Guppy: guppy upload <space> <source-path>
    Guppy->>Workers: ExecuteUpload(uploadID, spaceDID)
    Workers->>Workers: Scan FS entries
    Workers->>Workers: Create DAG nodes
    Workers->>Workers: Pack into CAR shards
    Workers->>Storacha: space/blob/add (shards)
    Workers->>Storacha: space/blob/add (indexes)
    Workers->>Storacha: upload/add
    Workers-->>Guppy: rootCID
    
    Note over Guppy,UCN: NEW: Mutability integration
    Guppy->>Pail: crdt.Put(sourcePath, rootCID)
    Pail-->>Guppy: eventCID
    Guppy->>UCN: name.Publish(eventCID)
    UCN-->>Guppy: OK
    Guppy-->>User: Uploaded (rootCID)
```

### Encrypted Upload with Mutability (Proposed)

```mermaid
sequenceDiagram
    participant User
    participant Guppy
    participant Workers
    participant Storacha
    participant KMS
    participant Pail
    participant UCN
    
    User->>Guppy: guppy upload --encrypt <space> <source-path>
    Guppy->>Guppy: Generate DEK
    Guppy->>Workers: ExecuteUpload with encryption
    Workers->>Workers: Encrypt blocks with DEK
    Workers->>Storacha: space/blob/add (encrypted shards)
    Workers->>Storacha: space/blob/add (indexes)
    Workers->>Storacha: upload/add
    Workers-->>Guppy: encryptedRootCID
    
    Guppy->>KMS: Wrap DEK with KEK
    KMS-->>Guppy: wrappedDEK
    Guppy->>Guppy: Create metadata block (wrappedDEK + encryptedRootCID)
    Guppy->>Storacha: space/blob/add (metadata block)
    Storacha-->>Guppy: metadataCID
    
    Note over Guppy,UCN: Mutability stores metadataCID
    Guppy->>Pail: crdt.Put(sourcePath, metadataCID)
    Pail-->>Guppy: eventCID
    Guppy->>UCN: name.Publish(eventCID)
    UCN-->>Guppy: OK
    Guppy-->>User: Uploaded (encrypted)
```

### Retrieve with Mutability - Unified Flow (Proposed)

```mermaid
sequenceDiagram
    participant User
    participant Guppy
    participant UCN
    participant Pail
    participant Locator
    participant Storage
    participant KMS
    
    User->>Guppy: guppy retrieve <space> <path> <output>
    
    alt Path is CID (e.g. bafy...)
        Guppy->>Guppy: Use CID directly as rootCID
    else Path is file path (e.g. /backups/db.tar)
        Note over Guppy,Pail: NEW: Resolve path via UCN + Pail
        Guppy->>UCN: name.Resolve(spaceDID)
        UCN-->>Guppy: head events
        Guppy->>Storage: Fetch Pail blocks for head
        Storage-->>Guppy: Pail blocks
        Guppy->>Pail: crdt.Root(head, blocks)
        Pail-->>Guppy: pailRoot
        Guppy->>Pail: pail.Get(pailRoot, path)
        Pail-->>Guppy: rootCID
    end
    
    Guppy->>Locator: Query indexer for rootCID
    Locator-->>Guppy: provider locations
    Guppy->>Storage: Fetch root block
    Storage-->>Guppy: block data
    
    alt Block is EncryptedMetadata format
        Note over Guppy,KMS: Encrypted content detected
        Guppy->>Guppy: Extract wrappedDEK + encryptedRootCID
        Guppy->>KMS: Unwrap DEK with KEK
        KMS-->>Guppy: DEK
        Guppy->>Locator: Query indexer for encryptedRootCID
        Locator-->>Guppy: provider locations
        Guppy->>Storage: Fetch encrypted blocks
        Storage-->>Guppy: encrypted blocks
        Guppy->>Guppy: Decrypt with DEK
    else Block is plaintext UnixFS
        Note over Guppy,Storage: Plaintext content
        Guppy->>Storage: Fetch remaining blocks
        Storage-->>Guppy: file blocks
    end
    
    Guppy-->>User: Write file to output
```

## References

- [Mutability & Privacy in Storacha — Strategy Document](https://www.notion.so/storacha/Mutability-Privacy-in-Storacha-Strategy-Document-3125305b5524807fb4a1ce6a3c9201e8) (internal)
- [Storacha UCN Package](https://github.com/storacha/upload-service/tree/main/packages/ucn)
- [Storacha Pail Package](https://github.com/storacha/pail)
- [UCAN Specification](https://github.com/ucan-wg/spec)
