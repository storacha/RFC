# RFC: Encryption for Forge (Guppy/Piri)

**Status: Draft**

## Authors

- [Felipe Forbeck](https://github.com/fforbeck), [Storacha Network](https://storacha.network/)

## Editors
- [Alan Shaw](https://github.com/alanshaw), [Storacha Network](https://storacha.network/)
- [Hannah Howard](https://github.com/hannahhoward), [Storacha Network](https://storacha.network/)
- [Alex Kinstler](https://github.com/prodalex), [Storacha Network](https://storacha.network/)

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Introduction

This RFC proposes the implementation of content encryption for Forge enterprise customers using the Guppy client. This enables:

1. **Encryption at Rest**: Client-side encryption of file content before upload
2. **UCAN-Gated Decryption**: Access control via delegations

**Scope:** File-level encryption only. Directory/folder encryption is not supported (each file is encrypted individually with its own DEK).

## Motivation

Enterprise Forge customers need:

- **Data protection**: Sensitive content must be encrypted before leaving the client
- **Key management**: Integration with Storacha KMS or customer-managed KMS
- **Access control**: Fine-grained control over who can decrypt content

---

## Industry Context

The proposed encryption model encrypts **file content only**, leaving metadata (paths, filenames, sizes) visible. This is the standard approach used by major cloud storage providers:

| Provider | Content Encrypted | Paths/Names Encrypted |
|----------|-------------------|----------------------|
| AWS S3 (SSE-S3, SSE-KMS) | ✅ | ❌ |
| Azure Blob Storage | ✅ | ❌ |
| Google Cloud Storage | ✅ | ❌ |

## Implementation Status

Based on Alan's POC ([storacha/guppy#376](https://github.com/storacha/guppy/pull/376)).

| Component | Status | Notes |
|-----------|--------|-------|
| AES-256-CTR encryption | ✅ Done | `pkg/encryption/aes_256_ctr.go` |
| Random IV per block | ✅ Done | Generated in `EncryptAES256CTR()` |
| IV stored in metadata | ✅ Done | `pkg/preparation/dags/nodemeta/aes_256_ctr.go` |
| Block-level splitter | ✅ Done | `pkg/preparation/dags/aes_256_ctr_splitter.go` |
| Decryption | ✅ Done | `DecryptAES256CTR()` function |
| KMS integration | ❌ Pending | `space/encryption/setup`, key unwrap |
| Metadata format | ❌ Pending | `EncryptedMetadata` CBOR block |
| UCAN delegation | ❌ Pending | `space/content/decrypt` handling |
| Folder access control | ❌ Pending | `nb.prefix` validation |

## Terminology

| Term | Description |
|------|-------------|
| **DEK** | Data Encryption Key 256-bit AES key used to encrypt file content |
| **KEK** | Key Encryption Key space-level RSA key managed by KMS, used to wrap DEKs |
| **IV** | Initialization Vector 16-byte random value, unique per content block |
| **Content block** | A chunk of encrypted file data (e.g., 1MB), contains IV inline |
| **Encrypted metadata block** | CBOR block containing wrapped DEK, path, KMS info. One per file, its CID is the "metadataCID" |
| **Wrapping** | Encrypting a DEK with the space's KEK (RSA-OAEP) |

## Encryption vs Access Control Granularity

| Aspect | Granularity | Description |
|--------|-------------|-------------|
| **Encryption (DEK)** | Per file | Each file has its own DEK |
| **Key wrapping (KEK)** | Per space | All DEKs in a space are wrapped with the same KEK |
| **Access control** | Per file OR per folder | CID-based (`nb.resource`) or path-based (`nb.path`) |

**Key insight:** Even though encryption is file-level, access control can be folder-level. A delegation with `nb.path: "/backups/"` grants access to decrypt **all files** whose path starts with `/backups/`, each with their own DEK.

## Encryption Approach: Block-Level

### How It Works

Block-level encryption combines file-scoped DEKs with block-scoped IVs:

- **1 DEK per file**: Each file gets a unique 256-bit AES key, generated randomly
- **Random IV per block**: Each block within the file gets a unique 16-byte IV
- **AES-256-CTR**: Counter mode encryption with unique keystream per block. AES-256-CTR doesn't provide authenticated encryption on its own, but in our case its fine since we use CIDs. Any tampering with the encrypted data changes the block's CID, which propagates up and changes the root CID, so it won't go unnoticed.
- **Incremental uploads**: Only changed blocks need re-encryption

### DEK Lifecycle (Per File)

```
For each file to upload:

1. Generate random DEK (256-bit AES key)
        │
        ▼
2. For each block in file:
   - Generate random IV (16 bytes)
   - Encrypt block with DEK + IV
   - Store IV in block metadata
        │
        ▼
3. Wrap DEK with space's public key (RSA-OAEP)
   - "Wrapping" = encrypting the DEK with the space's KEK (Key Encryption Key)
        │
        ▼
4. Store wrapped DEK in encrypted metadata block
        │
        ▼
5. Upload encrypted blocks + metadata
```

**Key insight:** The DEK is the same for all blocks within a file, but each block has a unique IV. This allows:
- Incremental uploads (only changed blocks re-encrypted with same DEK)
- File-level access control (revoke access to specific files)
- TS client compatibility (same pattern as `@storacha/encrypt-upload-client`)

### Security: IV Requirements

**MUST**
- Generate a new random IV (16 bytes) for every block
- Never reuse an IV with the same DEK
- Use cryptographically secure random number generator for IV generation

These requirements apply to all encryption operations, including initial uploads and incremental re-uploads.

**Storage overhead:** Each block stores a 16-byte IV. For 1MB blocks, this is ~0.0015% overhead, negligible in practice.

### Why Block-Level (vs File-Level)

| Approach | Incremental Uploads | KMS Calls | DEK | IV |
|----------|---------------------|-----------|-----|-----|
| **File-level** | ❌ Re-upload entire file | 1 per file | 1 per file | 1 per file |
| **Block-level** | ✅ Only changed blocks | 1 per session | 1 per file | 1 per block |

Block-level encryption uses a single DEK per file (stored in the encrypted metadata block), but each block has its own IV (stored inline with the block). This enables Guppy's existing incremental upload capability to work with encrypted content.


## Metadata Format

Each encrypted **file** has its own encrypted metadata block (1 per file, not per upload). This allows file-level access control and independent key rotation.

Encrypted content MUST include an encrypted metadata block compatible with `@storacha/encrypt-upload-client`:

```typescript
interface EncryptedMetadata {
  encryptedDataCID: CID        // CID of encrypted content
  encryptedSymmetricKey: string // Base64-encoded wrapped blob (contains path + DEK)
  space: SpaceDID              // Space the content belongs to
  path?: string                // File path for client display (e.g., "/backups/db.tar")
  kms: {
    provider: string           // e.g., "storacha"
    keyId: string              // KMS key identifier
    algorithm: string          // e.g., "RSA-OAEP-256"
  }
}
```

**Wrapped blob format:** The `encryptedSymmetricKey` contains:
- `wrap(KEK, { path, dek })` — if path is provided
- `wrap(KEK, { dek })` — if no path (backward compatible)

**Note:** When `path` is provided, it appears in two places:
- **In encrypted metadata block** (plaintext CBOR): For client display and Pail indexing
- **In wrapped blob** (encrypted): For KMS validation, the client can't lie about the path

## Streaming Support

Encryption MUST support streaming for large files (1TB+) with O(1) memory usage.

Go's standard library provides native support via:
- `crypto/aes`: AES block cipher
- `crypto/cipher`: CTR mode stream cipher
- `io.Reader` wrapper pattern: encrypt/decrypt chunks on-the-fly without buffering entire file

## Encryption Flow

For encrypted sources, the flow is triggered by `guppy upload` after a source was registered with `--encrypt` (see [forge-mutability.md](./forge-mutability.md#guppy-upload-extended)):

```
1. guppy upload <space> [source-name...]
        │
        ▼
2. Get space public key from KMS (space/encryption/setup)
   → 1 KMS call per upload session
        │
        ▼
3. For each file:
   a. Generate random DEK (256-bit AES key)
   b. For each file chunk:
      - Generate random IV (16 bytes)
      - Encrypt chunk with DEK + IV (AES-256-CTR)
      - Store IV in block metadata
   c. Build UnixFS DAG from encrypted blocks
   d. Wrap DEK with space public key (RSA-OAEP)
   e. Create encrypted metadata block with wrapped DEK + file path
        │
        ▼
4. Upload encrypted blocks + metadata (blob/add)
```

**POC Status**
- Step 2 (KMS): ❌ Pending
- Step 3a (DEK generation): ❌ Pending (POC uses manually provided key)
- Step 3b (IV + encryption): ✅ Done
- Step 3c (UnixFS DAG): ✅ Done
- Step 3d (DEK wrapping): ❌ Pending
- Step 3e (encrypted metadata block): ❌ Pending
- Step 4 (upload): ✅ Existing Guppy functionality

## Decryption Flow

Two decryption modes are supported:

### Option A: Gateway-Side Decryption (Local Key Mode)

For local/development use, the gateway can decrypt content on-the-fly:

```bash
guppy gateway serve --decryption-key /path/to/key.bin
```

```
1. Client requests: http://localhost:3000/ipfs/<CID>
        │
        ▼
2. Gateway fetches encrypted blocks from network
        │
        ▼
3. Gateway decrypts each block using provided key
   - Read IV from block (first 16 bytes)
   - Decrypt with key + IV (AES-256-CTR)
        │
        ▼
4. Gateway serves decrypted content to client
```

**POC Status:** ✅ Done (`--decryption-key` flag implemented)

### Option B: Client-Side Decryption (KMS Mode)

For production with access control, decryption happens client-side via `guppy retrieve` (see [forge-mutability.md](./forge-mutability.md#guppy-retrieve-extended)):

```
1. guppy retrieve <space> <path-or-cid> <output> --delegation <file>
   - Fetches encrypted content via gateway or directly from network
        │
        ▼
2. Extract encrypted metadata block
        │
        ▼
3. Extract wrapped DEK from metadata
        │
        ▼
4. Unwrap DEK via KMS (space/encryption/key/decrypt)
   → Client sends wrapped DEK to KMS (KMS does NOT fetch content)
   → Provide UCAN proof with space/content/decrypt delegation
   → KMS validates nb.resource matches the metadata CID
        │
        ▼
5. For each encrypted block:
   - Read IV from block metadata
   - Decrypt with unwrapped DEK + IV (AES-256-CTR)
        │
        ▼
6. Reassemble file from decrypted blocks
```

**POC Status:**
- Step 1 (gateway fetch): ✅ Existing infrastructure
- Step 2-4 (metadata + KMS): ❌ Pending
- Step 5 (IV + decryption): ✅ Done
- Step 6 (reassemble): ✅ Standard UnixFS

## Folder-Level Access Control

The existing `space/content/decrypt` capability uses `nb.resource` (CID-based). We propose enhancing it with an optional `nb.path` field for path-based access control:

```typescript
space/content/decrypt
  with: did:key:zSpace
  nb: { 
    resource: CID,              // Required: CID of the encrypted metadata block
    path: "/backups/"           // Optional: directory path (must end with /)
  }
  audience: did:key:zRecipient
```

**Validation flow**

1. User requests decryption with `nb.resource` (the metadata CID)
2. KMS validates `nb.resource`:
   - `invocation.nb.resource === delegation.nb.resource`
3. KMS unwraps the encrypted blob to get `{ path, dek }`
   - The `path` is cryptographically bound to the DEK at encryption time
   - KMS doesn't need to fetch content — path is embedded in the wrapped blob
4. If `nb.path` present in delegation, KMS validates:
   - Check: `path.startsWith(delegation.nb.path)`
   - `nb.path` MUST end with `/` to ensure directory matching (e.g., `/priv/` not `/priv`)
5. If all validations pass → return DEK; otherwise → reject

**Access control examples**

| `nb.resource` | `nb.path` | File `path` | Access |
|---------------|-----------|-------------|--------|
| `bafy...abc` | (none) | `/backups/db.tar` | ✅ CID-only access |
| `bafy...abc` | `/backups/` | `/backups/db.tar` | ✅ Path under directory |
| `bafy...abc` | `/priv/` | `/priv/secret.txt` | ✅ Path under directory |
| `bafy...abc` | `/priv/` | `/priv.txt` | ❌ Not under /priv/ directory |
| `bafy...abc` | `/logs/` | `/backups/db.tar` | ❌ Path mismatch |
| `bafy...abc` | `/backups/` | (none) | ❌ No path in file metadata |

**Backward compatibility**

If a delegation specifies `nb.path`, the file's metadata MUST contain a `path` field to validate against. Old files without `path` in metadata cannot be accessed using path-scoped delegations, so use a CID-only delegation instead (like we already do today).

**Design decision:** `nb.path` is a single string, not an array. To grant access to multiple paths, create separate delegations. This enables granular revocation. Revoking access to one path doesn't affect others.

## Capabilities Needed

| Capability | Status | Notes |
|------------|--------|-------|
| `space/encryption/setup` | Exists | Get space public key for wrapping |
| `space/encryption/key/decrypt` | Exists | Unwrap space DEK |
| `space/content/decrypt` | Exists | Authorization proof for decryption |

## Key Management

Guppy SHOULD support two key management modes.

### Local Key Mode (Development/Testing)

For development and testing, Guppy MAY use a locally-provided key. This can be configured either:

**Via `upload source add` flag:**
```bash
guppy upload source add <space> <path> --name <alias> --local-key ./dev-key.bin
```

**Or via config.yaml:**
```yaml
# ~/.storacha/guppy/config.yaml
encryption:
  enabled: true
  mode: local
  key: "base64-encoded-256-bit-key"  # Or path to key file
```

This mode does NOT provide access control — anyone with the key can decrypt.

See [guppy#376](https://github.com/storacha/guppy/pull/376) for the POC implementation.

### KMS Mode (Production/Enterprise)

For production, Guppy SHOULD use the Storacha KMS:

```yaml
encryption:
  enabled: true
  mode: kms
  kms_url: "https://kms.storacha.network"
  kms_did: "did:web:kms.storacha.network"
```

| Config | Description |
|--------|-------------|
| `UCAN_KMS_URL` | ucan-kms service endpoint |
| `UCAN_KMS_DID` | ucan-kms service DID for UCAN audience |

KMS mode enables:
- UCAN-gated decryption
- Folder-level access control via `nb.prefix`
- Key rotation (see below)

## Key Rotation

Key rotation requires the **Mutability** feature (see [forge-mutability.md](./forge-mutability.md)) because:
- Rotation creates new metadata CIDs (wrapped DEK changes)
- Pail entries must be updated to point to the new metadata CID
- UCN publishes the updated Pail head so clients can discover the rotated metadata

Guppy SHOULD support two types of key rotation via CLI commands (KMS mode only):

### KEK Rotation (Space Key)

Rotates the space-level Key Encryption Key without re-encrypting content:

```bash
guppy encryption rotate-kek --space <space-did>
```

**Process:**
1. Generate new KEK in KMS
2. List encrypted files using `pail.Entries()` (see [go-pail](https://github.com/storacha/go-pail))
   - Each encrypted file has one Pail entry: `path → metadataCID`
   - Since encryption is file-level only, each entry corresponds to one encrypted file
3. For each encrypted file in space:
   - Unwrap DEK with old KEK
   - Re-wrap DEK with new KEK
   - Create new encrypted metadata block (new CID)
   - Update Pail entry to point to new metadata CID
4. Delete old KEK from KMS
5. Content blocks remain unchanged

**Security note:** Old encrypted metadata blocks remain on the network (IPFS is immutable), but the old wrapped DEKs inside them are useless. The old KEK is deleted from KMS, so they cannot be unwrapped.

**Use case:** Regular security hygiene, suspected KEK compromise

### DEK Rotation (File Key)

Rotates a file's Data Encryption Key by re-encrypting the file:

```bash
guppy encryption rotate-dek --file <path>
```

**Process:**
1. Download and decrypt file with old DEK
2. Generate new DEK
3. Re-encrypt all blocks with new DEK + new IVs
4. Create new encrypted metadata block with new wrapped DEK
5. Upload encrypted blocks + new metadata
6. Update catalog entry to point to new root CID

**Use case:** Suspected file-level compromise, compliance requirements

### Local Key Mode

Key rotation is NOT supported in local key mode:
- No catalog/UCN to track metadata CID changes
- Users managing their own keys are responsible for rotation

## Revocation

Access revocation is handled by the `ucan-kms` service:

1. User revokes a `space/content/decrypt` delegation by CID
2. KMS checks revocation status on every decrypt request
3. Revoked delegations are rejected — DEK is not unwrapped

**Guppy CLI:**

```bash
guppy delegation revoke <delegation-cid>
```

**Note:** Revocation only prevents future decryption. If a user already has the DEK in memory, they can still decrypt. For full revocation, combine with DEK rotation. *(This behavior should be documented in user-facing docs)*

## Implementation Requirements

| Component | Change |
|-----------|--------|
| `@storacha/capabilities` (TS) | Add `prefix` field to `space/content/decrypt` schema |
| `go-libstoracha` (Go) | Port `prefix` field to Go capabilities |
| `ucan-kms` | Validate prefix in decrypt handler |
| `guppy` | CLI command to mint scoped delegations |


## Security Considerations

1. **Client-side encryption**: Plaintext MUST never leave the Guppy client
2. **Key management**: DEK is wrapped with space-specific KEK managed by KMS
3. **UCAN-gated decryption**: `space/encryption/key/decrypt` delegation required
4. **Unique IV per block**: Each block MUST have a unique random IV
5. **Secure random**: IVs MUST be generated using cryptographically secure random
6. **No IV reuse**: Same DEK with same IV = keystream reuse attack


## References

- [Block Encryption POC](https://github.com/storacha/guppy/pull/376)
- [Storacha Encrypt Upload Client](https://github.com/storacha/upload-service/tree/main/packages/encrypt-upload-client)
- [AWS S3 Client-Side Encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingClientSideEncryption.html)