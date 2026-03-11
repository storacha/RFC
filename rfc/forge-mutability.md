# RFC: Mutability for Forge (Guppy/Piri)

**Status: Draft — Pending Alignment**

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
2. **Content Catalog**: Track uploaded content (path → CID mappings) across clients

**Related:** [forge-encryption.md](./forge-encryption.md) — Encryption key rotation depends on mutability to track metadata CID changes.

## Motivation

Enterprise Forge customers need:

- **Mutable references**: Backup workflows need stable names that update to point to the latest backup version
- **Cross-client sync**: Multiple Guppy instances should be able to resolve the same reference
- **On-network state**: Track what has been uploaded without relying on local-only databases

## Approaches Under Consideration

### Option A: Simple Catalog (Alex's Proposal)

**How it works**

- **Namespace:** Use the Space DID directly, no new naming system needed
- **Catalog:** A simple CBOR file with sorted entries mapping `path → CID`. Chunked for large spaces.
- **Mutability:** Use `clock/head` / `clock/advance` to point to current catalog CID (needs Go impl)
- **Multi-writer:** Optimistic retry: if conflict, re-read catalog and retry

**What to build**
- Catalog format (CBOR, sorted entries, chunked for large spaces)
- `guppy upload` builds/updates catalog after uploading files
- `guppy ls` resolves catalog, lists entries
- `guppy gateway <path>` resolves catalog, finds entry, fetches content

**What NOT to build**
- Pail
- UCN (not needed, space DID is the namespace)
- Merkle clock CRDT merge (not needed for CLI tool)
- Go port of any TS package (build Forge-native)

**Catalog Format**

The catalog is a CBOR-encoded block with sorted entries:

```typescript
interface Catalog {
  version: 1
  entries: CatalogEntry[]
}

interface CatalogEntry {
  path: string           // File path (e.g., "/backups/server1/backup.tar")
  root: CID              // Entry point CID:
                         //   - Encrypted files: metadata block CID (contains wrapped DEK + link to encrypted content)
                         //   - Plaintext files: content root CID
  size: number           // File size in bytes
  encrypted?: boolean    // True if content is encrypted
  updated: number        // Unix timestamp of last update
}
```

**Chunking for large catalogs**
- If catalog exceeds 1MB, split into chunks
- Root catalog block contains links to chunk blocks
- Each chunk contains a sorted subset of entries

```typescript
interface ChunkedCatalog {
  version: 1
  chunks: CID[]          // Links to CatalogChunk blocks
  totalEntries: number
}

interface CatalogChunk {
  entries: CatalogEntry[]
  startPath: string      // First path in this chunk (for binary search)
  endPath: string        // Last path in this chunk
}
```

**Capabilities needed**

| Capability | Status | Notes |
|------------|--------|-------|
| `clock/head` | Needs Go impl | Read catalog pointer (exists in TS, not in go-libstoracha) |
| `clock/advance` | Needs Go impl | Update catalog pointer (exists in TS, not in go-libstoracha) |
| `blob/add` | Exists | Upload catalog + content |

**Trade-offs**

| Aspect | Option A (Simple Catalog) | Option B (CRDT) |
|--------|---------------------------|-----------------|
| **Concurrency** | Last write wins | Automatic merge |
| **Use case** | Single writer (CLI) | Multi-writer (teams) |
| **Complexity** | Low (CBOR list) | High (Pail + CRDT) |
| **Implementation** | Native Go | Port TS libraries |
| **Catalog size** | Works for 100k+ files | Optimized for millions |
| **Conflict resolution** | Manual (user re-uploads) | Automatic (CRDT merge) |

**Recommendation:** Start with Option A for Guppy CLI (single-user tool). Option B becomes valuable when enabling team collaboration with concurrent uploads from multiple clients.

### Option B: Go Ports (Hannah's Proposal)

**How it works**

- **Namespace:** UCN Names - ed25519 keypairs that can be delegated and shared
- **State Index:** Pail - sharded Merkle trie for `path → CID` mappings  
- **Concurrency:** Merkle clock CRDT enables automatic merge of concurrent writes

**What to build**
- UCN Go port (Name creation, publish, resolve, grant)
- Pail Go port (put, get, del, entries, diff, merge)

**What NOT to build**
- Service layer (all CID generation must happen on client)

**Capabilities needed**

| Capability | Status | Notes |
|------------|--------|-------|
| `clock/head` | Needs Go impl | Read current value of a Name (exists in TS) |
| `clock/advance` | Needs Go impl | Publish new value to a Name (exists in TS) |
| `blob/add` | Exists | Upload content |

## Comparison

| Aspect | Option A | Option B |
|--------|-----------------|-------------------|
| **Namespace** | Space DID (already exists) | UCN Name (new keypair per name) |
| **State Index** | CBOR catalog file | Pail trie (content-addressed KV) |
| **Mutability** | `clock/head` + `clock/advance` | UCN + Merkle clock |
| **Multi-writer** | Last-writer-wins + retry | CRDT merge (concurrent edits merge) |
| **TS ecosystem compat** | No | Yes |
| **Go ports needed** | None | UCN + Pail |

## Key Questions for Alignment

1. **Do Forge customers need fine-grained multi-writer?**
   - If yes (real-time collaboration), then Option B
   - If no (backup/archival, single writer), then Option A

2. **Do we need TypeScript client compatibility?**
   - If yes (Console, w3up-client interop), then Option B
   - If no (Forge is standalone), then Option A

## References

- [Mutability & Privacy in Storacha — Strategy Document](https://www.notion.so/storacha/Mutability-Privacy-in-Storacha-Strategy-Document-3125305b5524807fb4a1ce6a3c9201e8) (internal)
- [Storacha UCN Package](https://github.com/storacha/upload-service/tree/main/packages/ucn)
- [Storacha Pail Package](https://github.com/storacha/pail)
- [UCAN Specification](https://github.com/ucan-wg/spec)
