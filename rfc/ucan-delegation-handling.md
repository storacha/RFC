# RFC: Standardizing UCAN Delegation Handling

## Summary

This RFC proposes standardizing how we handle UCAN delegations and proofs across our services, adopting CAR (Content-Addressed aRchive) files as the canonical format with base64 encoding for transport.

## Problem Statement

Our services currently accept UCAN delegations in multiple formats:

- Base64 encoded strings.

- Base64-encoded CIDv1 with embedded CAR data.

This fragmentation creates:

- **Integration complexity**: Different services expect different formats
- **Debugging difficulty**: Multiple encoding layers obscure issues
- **Documentation burden**: Each format requires separate examples
- **Tooling gaps**: No standard way to validate or inspect delegations

## Proposed Solution: CAR File Standard

- **Storage format**: Binary (CBOR) CAR files with `.car` extension
- **Transport encoding**: Base64 encoded `.car` content for environment variables and HTTP headers

### Why CAR?

1. **Ecosystem compatibility**: Leverage existing tooling.
2. **Binary efficiency**: Compact storage for delegation data.
3. **Standard tooling**: `ipfs-car`, `ipfs` CLI support.

### Usage Examples

#### File-based configuration:

```bash
# Place delegation in standard location
cp delegation.car /etc/service/delegation.car

# Inspect CAR Roots
ipfs-car roots delegation.car

# Inspect CAR Blocks
ipfs-car blocks delegation.car
```

#### Environment Variable Configuration:

```bash
# Convert CAR to base64 for environment
export UCAN_DELEGATION=$(base64 -w 0 delegation.car)

# In Docker/Kubernetes
env:
  - name: UCAN_DELEGATION
    value: "OqJlcm9vdHOB2CpYJQAB..."
```

#### HTTP header usage:

```bash
# Include delegation in request
curl -H "Authorization: UCAN $(base64 -w 0 delegation.car)" \
     https://api.example.com/resource

# Alternative header format
curl -H "X-UCAN-Delegation: $(base64 -w 0 delegation.car)" \
     https://api.example.com/resource

```

### Implementation Guidelines

```go
func loadDelegation(config Config) ([]byte, error) {
    // Priority 1: File path (recommended)
    if config.DelegationFile != "" {
        return os.ReadFile(config.DelegationFile)
    }
    
    // Priority 2: Environment variable (containerized environments)
    if encoded := os.Getenv("UCAN_DELEGATION"); encoded != "" {
        return base64.StdEncoding.DecodeString(encoded)
    }
    
    // Priority 3: Direct value (testing only)
    if config.DelegationBase64 != "" {
        return base64.StdEncoding.DecodeString(config.DelegationBase64)
    }
    
    return nil, errors.New("no delegation provided")
}
```

### Standard Locations

- File storage: `/etc/service/delegation.car`
- Environment variable: `UCAN_DELEGATION`
- HTTP header: `Authorization: UCAN <base64>` or `X-UCAN-Delegation: <base64>`

## Looking Forward

In the future, services may accept just the root CID of a delegation CAR file instead of the full content.

### Benefits

1. **Minimal transport**: Pass single CIDs instead of kilobytes of data.
2. **De-duplication**: Common delegations stored once, referenced many times.
3. **Flexible retrieval**: Services choose optimal resolution strategy.

### Usage Examples

**Environment variable with CID**:

```bash
# Instead of full CAR file
export UCAN_DELEGATION_CID="bafy..."
```

**HTTP header with CID**:

```bash
# Lightweight authorization header
curl -H "Authorization: UCAN-CID bafy..." \
     https://api.example.com/resource
```

**Configuration hierarchy**:

```yaml
delegation:
  # Option 1: Local file
  file: /etc/service/delegation.car
  
  # Option 2: Inline base64
  data: "eyJyb290cyI6W3siLyI6ImJhZnlyZW..."
  
  # Option 3: CID reference (future)
  cid: "bafy..."
  resolver: "gateway.storacha.network"  # or "local-ipfs" or "s3://bucket/delegations"
```

### Resolution Strategies

Services receiving CIDs would need to implement resolution:

```go
func resolveDelegationCID(cid string, config ResolverConfig) ([]byte, error) {
    switch config.Type {
    case "storacha-gateway":
        return fetchFromGateway(config.Gateway, cid)
    
    case "local-ipfs":
        return fetchFromLocalIPFS(cid)
    
    case "s3":
        return fetchFromS3(config.Bucket, cid)
    
    case "delegation-service":
        return fetchFromDelegationService(config.Endpoint, cid)
    
    default:
        return nil, fmt.Errorf("unknown resolver type: %s", config.Type)
    }
}
```