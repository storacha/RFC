# RFC: Standardizing Private Key Material Handling

## Summary

This RFC proposes standardizing how we handle private key material across our services, adopting PEM format as the single standard for private key storage and configuration.

## Problem Statement

Our services currently accept private keys in multiple formats:

- PEM files (with inconsistent encryption practices).
- JSON files (non-standard, service-specific schemas).
- Base64 encoded strings (raw key material, no metadata).
- Multibase base64 padded strings (uncommon in standard tooling).

This creates:

- **Security inconsistency**: Different formats have varying security properties.
- **Implementation complexity**: Each service implements multiple parsers.
- **Cognitive overhead**: Developers must remember which format works where.
- **Error-prone deployments**: Format mismatches cause configuration failures.

## Proposed Solution: PEM Standard

**Format**: PEM-encoded private keys using PKCS#8 format

```
-----BEGIN PRIVATE KEY-----
MC4CAQA...
-----END PRIVATE KEY-----
```

### Why PEM?

1. **Universal tooling support**: OpenSSL, ssh-keygen, and all major crypto libraries support PEM natively.
2. **Self-documenting format**: Clear headers (`-----BEGIN PRIVATE KEY-----`) identify content type.
3. **Algorithm agnostic**: Supports our current Ed25519 keys and future algorithms (RSA, ECDSA, etc.).
4. **Built-in security**: Native support for passphrase encryption.
5. **Human readable**: Base64 encoded with clear delimiters for debugging.
6. **Industry standard**: Developers already understand `.pem` files.

### Usage Examples

**Generating Ed25519 keys**:

```bash
# Generate Ed25519 private key
openssl genpkey -algorithm ed25519 -out service-key.pem

# View key details
openssl pkey -in service-key.pem -text -noout

# Extract public key
openssl pkey -in service-key.pem -pubout -out service-pubkey.pem
```

**Securing keys**:

```bash
# Set proper permissions
chmod 600 service-key.pem

# Encrypt for storage
openssl pkey -in service-key.pem -out service-key-encrypted.pem -aes256

# Decrypt when needed
openssl pkey -in service-key-encrypted.pem -out service-key.pem
```

**Future algorithm support** (no code changes needed):

```bash
# RSA
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:4096 -out rsa-key.pem

# ECDSA
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -out ecdsa-key.pem
```

## Security Requirements

1. **File permissions**: Private key files MUST have 600 permissions.
2. **No environment variables**: Private keys MUST NOT be passed via environment variables.
3. **Encryption at rest**: Keys SHOULD be encrypted when stored.
4. **Memory handling**: Keys SHOULD be loaded once at startup, not repeatedly read.
5. **Secure deletion**: Key material SHOULD be securely overwritten when removed from memory.

## Implementation Guidelines

### Service Configuration

```go
type Config struct {
    PrivateKeyPath string `flag:"private-key" env:"-" description:"Path to PEM-encoded private key"`
}

func loadPrivateKey(path string) (crypto.PrivateKey, error) {
    // Check file permissions
    info, err := os.Stat(path)
    if err != nil {
        return nil, fmt.Errorf("accessing private key file: %w", err)
    }
    if info.Mode().Perm() != 0600 {
        return nil, fmt.Errorf("private key file must have 600 permissions, has %v", info.Mode().Perm())
    }
    
    pemData, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("reading private key: %w", err)
    }
    
    block, _ := pem.Decode(pemData)
    if block == nil {
        return nil, errors.New("failed to parse PEM block")
    }
    
    // Require unencrypted keys for services
    // NB: depending on the context this method is called in, we may prompt for passphrase.
    if x509.IsEncryptedPEMBlock(block) {
        return nil, errors.New("encrypted private keys are not supported; " +
            "use file permissions (600) for security instead")
    }
    
    // Parse unencrypted PKCS#8 private key
    privateKey, err := x509.ParsePKCS8PrivateKey(block.Bytes)
    if err != nil {
        // Fallback for legacy formats
        if key, err := x509.ParsePKCS1PrivateKey(block.Bytes); err == nil {
            privateKey = key
        } else if key, err := x509.ParseECPrivateKey(block.Bytes); err == nil {
            privateKey = key
        } else {
            return nil, fmt.Errorf("parsing private key: %w", err)
        }
    }
    
    // Validate and return supported key types
    // NB: at present we only support ed25519
    switch key := privateKey.(type) {
    case ed25519.PrivateKey:
        return key, nil
    default:
        return nil, fmt.Errorf("unsupported private key type: %T", privateKey)
    }
}
```

### Standard File Locations

- Development: `./keys/service-key.pem`
- Production: `/etc/service/keys/service-key.pem`
- Container: `/run/secrets/service-key`

## Benefits

- **Enhanced security**: Consistent security model across all services.
- **Reduced complexity**: Single format reduces code and documentation.
- **Better tooling**: Leverage decades of OpenSSL development.
- **Future proof**: Algorithm changes require no format changes.
- **Developer friendly**: Well-understood format with extensive documentation.

## DID Derivation from Private Keys

Users often need to derive a DID (Decentralized Identifier) from the private key for identity purposes. This section briefly covers how to extract the DID from a PEM-encoded private key.

### What is a DID and How do we Encode ours?

A DID is a decentralized identifier that corresponds to your public key. In our system, Ed25519 keys produce DIDs in the format: `did:key:...` where the suffix is a Base58BTC-encoded representation of the public key with multicodec prefixes.

### Deriving DID from PEM

At present, there does not exist any standard tooling to derive a DID used by our system from a PEM File containing a private key, or public key for that matter. This necessitates the creation of a custom CLI tool which allows users to derive a DID string from their public key.

