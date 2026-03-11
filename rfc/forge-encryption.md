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

## Encryption Approach: Block-Level

### How It Works

Block-level encryption combines file-scoped DEKs with block-scoped IVs:

- **1 DEK per file**: Each file gets a unique 256-bit AES key, generated randomly
- **Random IV per block**: Each block within the file gets a unique 16-byte IV
- **AES-256-CTR**: Counter mode encryption with unique keystream per block
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
        │
        ▼
4. Store wrapped DEK in file metadata block
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

### Why Block-Level (vs File-Level)

| Approach | Incremental Uploads | KMS Calls | Metadata |
|----------|---------------------|-----------|----------|
| **File-level** | ❌ Re-upload entire file | 1 per file | 1 per file |
| **Block-level** | ✅ Only changed blocks | 1 per session | IV per block |

Block-level enables Guppy's existing incremental upload capability to work with encrypted content.


## Metadata Format

Encrypted content MUST include a metadata block compatible with `@storacha/encrypt-upload-client`:

```typescript
interface EncryptedMetadata {
  encryptedDataCID: CID        // CID of encrypted content
  encryptedSymmetricKey: string // Base64-encoded wrapped DEK
  space: SpaceDID              // Space the content belongs to
  path?: string                // File path (e.g., "/backups/server1/backup.tar")
  kms: {
    provider: string           // e.g., "storacha"
    keyId: string              // KMS key identifier
    algorithm: string          // e.g., "RSA-OAEP-256"
  }
}
```

The `path` field is RECOMMENDED for all new uploads. It enables folder-level access control via `nb.prefix` delegations (see Folder-Level Access Control section).

## Streaming Support

Encryption MUST support streaming for large files (1TB+) with O(1) memory usage.

Go's standard library provides native support via:
- `crypto/aes`: AES block cipher
- `crypto/cipher`: CTR mode stream cipher
- `io.Reader` wrapper pattern: encrypt/decrypt chunks on-the-fly without buffering entire file

## Encryption Flow

```
1. guppy upload <source>
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
   e. Create metadata block with wrapped DEK + file path
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
- Step 3e (metadata block): ❌ Pending
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

For production with access control, decryption happens client-side:

```
1. Fetch encrypted content via gateway
   - Public: https://w3s.link/ipfs/<CID>
   - Local: guppy gateway serve → http://localhost:3000/ipfs/<CID>
        │
        ▼
2. Extract file metadata block
        │
        ▼
3. Extract wrapped DEK from metadata
        │
        ▼
4. Unwrap DEK via KMS (space/encryption/key/decrypt)
   → Provide UCAN proof with space/content/decrypt delegation
   → KMS validates nb.prefix against file path
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

Access control is enforced via the `nb.prefix` caveat on `space/content/decrypt` delegations:

```typescript
// Grant access to all files under /backups/server1/
space/content/decrypt
  with: did:key:zSpace
  nb: { prefix: "/backups/server1/" }
  audience: did:key:zRecipient
```

**Validation flow**

1. User requests decryption with delegation containing `nb.prefix`
2. KMS extracts `path` from encrypted metadata block
3. KMS validates: `path.startsWith(delegation.nb.prefix)`
4. If valid → unwrap DEK; if invalid → reject

**Access control examples**

| Delegation `nb.prefix` | File `path` | Access |
|------------------------|-------------|--------|
| `/backups/` | `/backups/server1/backup.tar` | ✅ Allowed |
| `/backups/server1/` | `/backups/server1/backup.tar` | ✅ Allowed |
| `/backups/server2/` | `/backups/server1/backup.tar` | ❌ Denied |
| (none) | `/backups/server1/backup.tar` | ✅ Space-level access |

**Backward compatibility**

- Delegations without `nb.prefix` grant space-level access (all files)
- Files uploaded without `path` field are treated as root (`/`) and accessible with any space-level delegation

## Capabilities Needed

| Capability | Status | Notes |
|------------|--------|-------|
| `space/encryption/setup` | Exists | Get space public key for wrapping |
| `space/encryption/key/decrypt` | Exists | Unwrap space DEK |
| `space/content/decrypt` | Exists | Authorization proof for decryption |

## Key Management

Guppy SHOULD support two key management modes.

### Local Key Mode (Development/Testing)

For development and testing, Guppy MAY use a locally-provided key:

```yaml
# ~/.storacha/guppy/config.yaml
encryption:
  enabled: true
  mode: local
  key: "base64-encoded-256-bit-key"  # Or path to key file
```

This mode does NOT provide access control — anyone with the key can decrypt.

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
- The catalog/UCN must be updated to point to the new metadata CID
- Without mutability, clients cannot discover the rotated metadata

Guppy SHOULD support two types of key rotation via CLI commands (KMS mode only):

### KEK Rotation (Space Key)

Rotates the space-level Key Encryption Key without re-encrypting content:

```bash
guppy encryption rotate-kek --space <space-did>
```

**Process:**
1. Generate new KEK in KMS
2. For each encrypted file in space:
   - Unwrap DEK with old KEK
   - Re-wrap DEK with new KEK
   - Create new metadata block (new CID)
   - Update catalog entry to point to new metadata CID
3. Content blocks remain unchanged

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
4. Create new metadata block with new wrapped DEK
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

**Note:** Revocation only prevents future decryption. If a user already has the DEK in memory, they can still decrypt. For full revocation, combine with DEK rotation.

## Implementation Requirements

| Component | Change |
|-----------|--------|
| `@storacha/capabilities` | Add `prefix` field to `space/content/decrypt` schema |
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
