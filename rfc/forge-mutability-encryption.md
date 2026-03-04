# RFC: Mutability and Encryption for Forge (Guppy/Piri)

**Status: Proposed Standard**

## Authors

- [Felipe Forbeck](https://github.com/fforbeck), [Storacha Network](https://storacha.network/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Introduction

This RFC proposes the implementation of mutability and content encryption features for Forge enterprise customers using the Guppy client and Piri storage nodes. These features enable:

1. **Mutability**: Stable, human-readable names that point to the latest version of uploaded content
2. **Content Encryption**: Client-side encryption of file content before upload to the Storacha network
3. **On-Network State Index**: Pail-based key-value store for tracking uploaded content across clients

Both features are already implemented in the TypeScript ecosystem (`@storacha/ucn`, `@storacha/encrypt-upload-client`). This RFC proposes porting these capabilities to Go for use in Guppy and Piri.

## Motivation

Enterprise Forge customers need:

- **Encryption at rest**: Data protection for sensitive content stored on the network
- **Mutable references**: Backup workflows need stable names that update to point to the latest backup version
- **Cross-client sync**: Multiple Guppy instances should be able to resolve the same named reference
- **On-network state**: Track what has been uploaded without relying on local-only databases

## Scope

### In Scope

- Content encryption (file bytes are encrypted)
- Infra for folder-level access control via `nb.prefix` on decrypt delegations
- Mutable naming via UCN (User Controlled Names)
- On-network state index via Pail (key-value store)
- Go implementation for Guppy client
- Compatibility with existing TypeScript implementations

### Out of Scope

- Metadata/path encryption (filenames and directory structure remain visible)
- Lit Protocol integration (KMS-only encryption)
- Cryptographic folder isolation (different encryption keys per folder)

## Industry Context: Content-Only Encryption

The proposed encryption model encrypts **file content only**, leaving metadata (paths, filenames, sizes) visible. This is the standard approach used by major cloud storage providers:

| Provider | Content Encrypted | Paths/Names Encrypted |
|----------|-------------------|----------------------|
| AWS S3 (SSE-S3, SSE-KMS) | ✅ | ❌ |
| Azure Blob Storage | ✅ | ❌ |
| Google Cloud Storage | ✅ | ❌ |
| Dropbox | ✅ | ❌ |

This model satisfies most enterprise "encryption at rest" requirements while maintaining the ability to list and traverse files without decryption. Full metadata encryption (as seen in zero-knowledge services like Proton Drive) is significantly more complex and breaks standard tooling compatibility.

## Proposal

### 1. Content Encryption

#### 1.1 Encryption Model

Guppy MUST implement envelope encryption:

1. **Data Encryption Key (DEK) + IV**: A random 256-bit AES key and 128-bit IV generated per file
2. **Key Wrapping**: The combined DEK+IV is wrapped using RSA-OAEP with a space-specific public key from the KMS
3. **Metadata Block**: Wrapped key + KMS info stored as CBOR block alongside encrypted content

```
┌─────────────────────────────────────────────────────────────┐
│                    ENCRYPTION FLOW                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Source File + optional file metadata                     │
│          │                                                   │
│          ▼                                                   │
│  2. Generate DEK + IV                                        │
│          │                                                   │
│          ▼                                                   │
│  3. Encrypt file content (AES-256-CTR) → encrypted stream    │
│          │                                                   │
│          ▼                                                   │
│  4. KMS: Wrap DEK+IV with RSA-OAEP (KMS) → encryptedSymmetricKey   │
│          │                                                   │
│          ▼                                                   │
│  5. Encode encrypted stream as UnixFS DAG → encryptedDataCID │
│          │                                                   │
│          ▼                                                   │
│  6. Create metadata block (CBOR):                            │
│     - encryptedDataCID (from step 5)                         │
│     - encryptedSymmetricKey (from step 4)                    │
│     - space DID                                              │
│     - KMS provider info                                      │
│          │                                                   │
│          ▼                                                   │
│  7. Upload CAR with encrypted content + metadata block       │
│     → returns root CID (metadata block CID)                  │
│                                                              │
│  ─────────────── POST-UPLOAD (UCN + Pail) ───────────────    │
│                                                              │
│  8. Record in Pail: put(filePath, rootCID)                   │
│     → returns new Pail root CID                              │
│          │                                                   │
│          ▼                                                   │
│  9. Publish to UCN: Name.publish(pailRootCID)                │
│     → mutable name now points to updated index               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Steps 1-7**: Encryption and upload (per file)
**Steps 8-9**: Index update and publish (after upload completes)

#### 1.2 KMS Integration

Guppy interacts with the existing `ucan-kms` service via UCAN invocations:

**Encryption (key wrapping):**
1. Invoke `space/encryption/setup` to get the space's RSA public key
2. Wrap DEK+IV locally using RSA-OAEP with the public key
3. Store wrapped key in metadata block

**Decryption (key unwrapping):**
1. Create `space/content/decrypt` delegation with appropriate proofs
2. Invoke `space/encryption/key/decrypt` on ucan-kms service
3. Service validates delegation, unwraps DEK+IV, returns plaintext key
4. Decrypt content locally with unwrapped key

| UCAN Capability | Purpose |
|-----------------|---------|
| `space/encryption/setup` | Get space's RSA public key for wrapping |
| `space/encryption/key/decrypt` | Unwrap DEK using space's private key |
| `space/content/decrypt` | Authorization proof for decryption |

Guppy MUST be configured with:
- `UCAN_KMS_URL` — ucan-kms service endpoint
- `UCAN_KMS_DID` — ucan-kms service DID for UCAN audience

#### 1.3 Streaming Support

Encryption MUST support streaming for large files (1TB+) with O(1) memory usage.

Go's standard library provides native support for this via:
- `crypto/aes` — AES block cipher
- `crypto/cipher` — CTR mode stream cipher
- `io.Reader` wrapper pattern — encrypt/decrypt chunks on-the-fly without buffering entire file

#### 1.4 Metadata Format

The metadata block MUST be compatible with `@storacha/encrypt-upload-client`:

```typescript
interface KMSMetadata {
  encryptedDataCID: CID        // CID of encrypted content
  encryptedSymmetricKey: string // Base64-encoded wrapped DEK+IV
  space: SpaceDID              // Space the content belongs to
  path?: string                // File path (e.g., "/backups/server1/backup.tar")
  kms: {
    provider: string           // e.g., "storacha"
    keyId: string              // KMS key identifier
    algorithm: string          // e.g., "RSA-OAEP-256"
  }
}
```

The `path` field is RECOMMENDED for all new uploads. It enables folder-level access control via `nb.prefix` delegations (see §1.6). Files without `path` are treated as root (`/`) for access control purposes.

#### 1.5 Decryption Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    DECRYPTION FLOW                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ─────────────── RESOLVE (UCN + Pail) ───────────────────    │
│                                                              │
│  1. Resolve UCN Name → get current Pail root CID             │
│          │                                                   │
│          ▼                                                   │
│  2. Query Pail for filePath → get content root CID           │
│                                                              │
│  ─────────────── DECRYPT ────────────────────────────────    │
│                                                              │
│          │                                                   │
│          ▼                                                   │
│  3. Fetch CAR from gateway using content CID                 │
│          │                                                   │
│          ▼                                                   │
│  4. Extract metadata block from CAR (CBOR)                   │
│     - encryptedDataCID                                       │
│     - encryptedSymmetricKey                                  │
│     - KMS info                                               │
│          │                                                   │
│          ▼                                                   │
│  5. Extract encrypted content from CAR using encryptedDataCID│
│          │                                                   │
│          ▼                                                   │
│  6. KMS: Unwrap DEK+IV via UCAN proof (space/content/decrypt)│
│          │                                                   │
│          ▼                                                   │
│  7. Split combined key → DEK (256-bit) + IV (128-bit)        │
│          │                                                   │
│          ▼                                                   │
│  8. Decrypt content stream (AES-256-CTR)                     │
│          │                                                   │
│          ▼                                                   │
│  9. Extract embedded file metadata (optional)                │
│     → returns decrypted stream + file metadata               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

User provides **UCN name + file path** to retrieve content:
1. **Resolve** — UCN name → Pail root → file path → content CID
2. **Decrypt** — fetch, unwrap key, decrypt stream

Decryption requires a `space/content/decrypt` delegation proof.

#### 1.6 Folder-Level Access Control

Access control is enforced via the `nb.prefix` caveat on `space/content/decrypt` delegations:

```typescript
// Grant access to all files under /backups/server1/
space/content/decrypt
  with: did:key:zSpace
  nb: { prefix: "/backups/server1/" }
  audience: did:key:zRecipient
```

**Validation flow:**

1. User requests decryption with delegation containing `nb.prefix`
2. KMS extracts `path` from encrypted metadata block
3. KMS validates: `path.startsWith(delegation.nb.prefix)`
4. If valid → unwrap DEK; if invalid → reject

**Access control examples:**

| Delegation `nb.prefix` | File `path` | Access |
|------------------------|-------------|--------|
| `/backups/` | `/backups/server1/backup.tar` | ✅ Allowed |
| `/backups/server1/` | `/backups/server1/backup.tar` | ✅ Allowed |
| `/backups/server2/` | `/backups/server1/backup.tar` | ❌ Denied |
| (none) | `/backups/server1/backup.tar` | ✅ Space-level access |

**Backward compatibility:**

- Delegations without `nb.prefix` grant space-level access (all files)
- Files uploaded without `path` field are treated as root (`/`) and accessible with any space-level delegation

**Implementation requirements:**

| Component | Change |
|-----------|--------|
| `@storacha/capabilities` | Add `prefix` field to `space/content/decrypt` schema |
| `ucan-kms` | Validate prefix in decrypt handler |
| `encrypt-upload-client` | Store `path` in encrypted metadata (already done) |
| `guppy` | CLI command to mint scoped delegations |

### 2. Mutability (UCN)

#### 2.1 Overview

UCN (User Controlled Names) provides mutable references using:

- **Merkle Clock**: CRDT for conflict-free multi-writer updates
- **UCAN Delegation**: Access control for read/write permissions
- **Network Sync**: Publish/resolve via clock service

#### 2.2 Service Layer Approach

Rather than porting the full UCN implementation to Go, Guppy SHOULD use a service layer:

```
┌─────────────────────────────────────────────────────────────┐
│                    UCN SERVICE LAYER                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Guppy (Go) ──▶ UCN Service (TS) ──▶ Clock Service          │
│       │              │                                       │
│       │              ▼                                       │
│       │         Name.create()                                │
│       │         Name.publish(cid)                            │
│       │         Name.resolve() → cid                         │
│       │         Name.grant(recipient)                        │
│       │                                                      │
└─────────────────────────────────────────────────────────────┘
```

#### 2.3 UCN Capabilities

The following UCAN capabilities are used:

| Capability | Description |
|------------|-------------|
| `clock/head` | Read current value of a Name |
| `clock/advance` | Publish new value to a Name |

#### 2.4 Guppy Integration

After upload completion, Guppy MUST:

1. Obtain the root CID of the uploaded content
2. Invoke the UCN service to publish the new CID to the configured Name
3. Provide UCAN proofs authorizing the `clock/advance` capability

This updates the mutable name to point to the latest upload, enabling clients to discover the current version without out-of-band CID coordination.

#### 2.5 Name Resolution

To retrieve the latest version, Guppy MUST:

1. Invoke the UCN service to resolve the Name to its current CID
2. Provide UCAN proofs authorizing the `clock/head` capability
3. Fetch the content from the gateway using the resolved CID
4. Optionally decrypt if the content is encrypted

### 3. On-Network State Index (Pail)

#### 3.1 Overview

Pail is a content-addressed key-value store implemented as a sharded prefix trie (Merkle DAG). It enables:

- **Path → CID mapping**: Track which files have been uploaded and their CIDs
- **Cross-client sync**: Multiple Guppy instances can share the same index
- **Version history**: Every mutation produces a new root CID, preserving history

#### 3.2 How Pail Works

```
┌─────────────────────────────────────────────────────────────┐
│                    PAIL STRUCTURE                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Root CID (v3)                                               │
│       │                                                      │
│       ├── /backups/server1/2025-03-01.tar.gz → bafy...abc   │
│       ├── /backups/server1/2025-03-02.tar.gz → bafy...def   │
│       ├── /backups/server2/2025-03-01.tar.gz → bafy...ghi   │
│       └── ...                                                │
│                                                              │
│  Operations: put(key, cid), get(key), del(key), entries()   │
│                                                              │
│  Each mutation → new root CID (immutable history)           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 3.3 Pail + UCN Integration

Pail and UCN work together:

1. **Pail** stores the key-value index (path → CID mappings)
2. **UCN** provides a mutable name pointing to the current Pail root

```
┌─────────────────────────────────────────────────────────────┐
│                    PAIL + UCN                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  UCN Name: "my-backup-index"                                 │
│       │                                                      │
│       ▼                                                      │
│  Current Pail Root: bafy...xyz                               │
│       │                                                      │
│       ├── /server1/backup.tar → bafy...encrypted1           │
│       ├── /server2/backup.tar → bafy...encrypted2           │
│       └── ...                                                │
│                                                              │
│  Workflow:                                                   │
│  1. Resolve UCN → get current Pail root                     │
│  2. Query Pail for path → get content CID                   │
│  3. Fetch and decrypt content                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 3.4 Service Layer Approach

Like UCN, Pail SHOULD be accessed via a service layer rather than a full Go port.

After each upload, Guppy MUST:

1. Record the file path and content CID in the Pail index via the service API
2. Provide UCAN proofs authorizing the Pail mutation
3. Obtain the new Pail root CID from the service
4. Publish the new Pail root to the UCN Name (as described in §2.4)

This ensures the on-network index stays synchronized with local upload state.

## Security Considerations

1. **Client-side encryption**: Plaintext MUST never leave the Guppy client
2. **Key management**: DEKs are wrapped with space-specific KEKs managed by KMS
3. **UCAN-gated decryption**: `space/content/decrypt` delegation required to unwrap keys
4. **No key reuse**: Each file MUST use a unique DEK
5. **Secure random**: Keys and IVs MUST be generated using cryptographically secure random

## Compatibility

- **Metadata format**: MUST be compatible with `@storacha/encrypt-upload-client`
- **UCN protocol**: MUST be compatible with `@storacha/ucn`
- **Pail format**: MUST be compatible with `@web3-storage/pail`
- **Existing uploads**: Unencrypted uploads continue to work unchanged

## Future Work

- **Metadata encryption**: Optional path/filename encryption for higher privacy requirements
- **Offline-first Pail**: Local Pail operations with sync when online

## References

- [Mutability & Privacy in Storacha — Strategy Document](https://www.notion.so/storacha/Mutability-Privacy-in-Storacha-Strategy-Document-3125305b5524807fb4a1ce6a3c9201e8) (internal)
- [Storacha UCN Package](https://github.com/storacha/upload-service/tree/main/packages/ucn)
- [Storacha Encrypt Upload Client](https://github.com/storacha/upload-service/tree/main/packages/encrypt-upload-client)
- [Storacha Pail Package](https://github.com/storacha/pail)
- [AWS S3 Server-Side Encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html)
- [UCAN Specification](https://github.com/ucan-wg/spec)
