# RFC: Adopting Uber FX for Dependency Injection in Piri

**Author**: Forrest (@frrist)

**Date**: 2025-06-25 

**Status**: Draft

## Summary

This RFC proposes adopting Uber's FX dependency injection framework for the Piri project to address current pain points in dependency management, improve testability, and enhance maintainability as the codebase grows.

## Background

The Piri project currently uses manual dependency wiring throughout its codebase. As demonstrated in our proof-of-concept (see [`demofx`](https://github.com/frrist/demofx) repository), this approach leads to several challenges:

1. **Complex initialization sequences** - Dependencies must be created in exact order
2. **Propagation of constructor changes** - Adding a new dependency requires updating all constructor calls
3. **Manual lifecycle management** - Start/stop sequences must be carefully orchestrated
4. **Testing complexity** - Mocking dependencies requires extensive setup

## Current State Analysis

### Example 1: AWS Service Construction

The AWS service construction in `pkg/aws/service.go` shows extreme complexity with 434 lines of manual wiring:

```go
func Construct(cfg Config) (storage.Service, error) {
    // Manual creation of 15+ AWS service clients and stores
    blobStore := NewS3BlobStore(cfg.Config, cfg.BlobStoreBucket, formatKey, blobStoreOpts...)
    allocationStore := NewDynamoAllocationStore(cfg.Config, cfg.AllocationsTableName, cfg.DynamoOptions...)
    claimStore, err := delegationstore.NewDelegationStore(NewS3Store(...))
    ipniStore := NewS3Store(cfg.Config, cfg.IPNIStoreBucket, cfg.IPNIStorePrefix, cfg.S3Options...)
    chunkLinksTable := NewDynamoProviderContextTable(...)
    metadataTable := NewDynamoProviderContextTable(...)
    publisherStore := store.NewPublisherStore(ipniStore, chunkLinksTable, metadataTable, ...)
    // ... and many more
    
    // Complex conditional logic for optional features
    if cfg.SQSPDPPieceAggregatorURL != "" && cfg.CurioURL != "" {
        pdp, err := NewPDP(cfg)
        // ... error handling and more setup
    }
    
    // Manual option building
    opts := []storage.Option{
        storage.WithIdentity(cfg.Signer),
        storage.WithBlobstore(blobStore),
        storage.WithAllocationStore(allocationStore),
        // ... 20+ more options
    }
    
    return storage.New(opts...)
}
```

### Example 2: Lambda Function Complexity

Lambda functions in `cmd/lambda/*/main.go` suffer from heavyweight initialization:

```go
func makeHandler(cfg aws.Config) (http.Handler, error) {
    // This constructs EVERYTHING even if lambda only needs blob store
    service, err := aws.Construct(cfg)
    if err != nil {
        return nil, err
    }
    
    // Only uses 3 components out of 20+ created
    handler := blobs.NewBlobPutHandler(
        service.Blobs().Presigner(), 
        service.Blobs().Allocations(), 
        service.Blobs().Store()
    )
    return handler, nil
}
```

### Example 3: UCAN Server Setup

The UCAN server in `cmd/cli/serve/ucan.go` shows 329 lines of manual dependency wiring:

```go
func startServer(cmd *cobra.Command, _ []string) error {
    // Manual creation of 6 different datastores
    allocDs, err := leveldb.NewDatastore(allocsDir, nil)
    claimDs, err := leveldb.NewDatastore(claimsDir, nil)
    publisherDs, err := leveldb.NewDatastore(publisherDir, nil)
    receiptDs, err := leveldb.NewDatastore(receiptDir, nil)
    // ... plus conditional PDP datastore
    
    // Complex configuration parsing and validation
    uploadServiceDID, err := did.Parse(cfg.UploadServiceDID)
    uploadServiceURL, err := url.Parse(cfg.UploadServiceURL)
    indexingServiceDID, err := did.Parse(cfg.IndexingServiceDID)
    indexingServiceURL, err := url.Parse(cfg.IndexingServiceURL)
    // ... and many more
    
    // Manual option building with 20+ options
    opts := []storage.Option{
        storage.WithIdentity(id),
        storage.WithBlobstore(blobStore),
        storage.WithAllocationDatastore(allocDs),
        // ... extensive list
    }
    
    svc, err := storage.New(opts...)
}
```

### Example 4: PDP Server Initialization

The current PDP server initialization in `pkg/pdp/server.go` demonstrates the complexity:

```go
func NewServer(ctx context.Context, dataDir string, endpoint *url.URL, 
    lotusUrl string, address common.Address, wlt *wallet.LocalWallet) (*Server, error) {
    
    // 1. Create datastore
    ds, err := leveldb.NewDatastore(filepath.Join(dataDir, "datastore"), nil)
    
    // 2. Create blob store (depends on datastore)
    blobStore := blobstore.NewTODO_DsBlobstore(namespace.Wrap(ds, datastore.NewKey("blobs")))
    
    // 3. Create stash store
    stashStore, err := store.NewStashStore(path.Join(dataDir))
    
    // 4. Create chain client
    chainClient, chainClientCloser, err := client.NewFullNodeRPCV1(ctx, lotusURL.String(), nil)
    
    // 5. Create eth client
    ethClient, err := ethclient.Dial(lotusUrl)
    
    // 6. Create state database
    stateDB, err := gormdb.New(filepath.Join(stateDir, "state.db"), ...)
    
    // 7. Create PDP service (depends on ALL the above)
    pdpService, err := service.NewPDPService(stateDB, address, wlt, 
        blobStore, stashStore, chainClient, ethClient, &contract.PDPContract{})
        
    // Manual lifecycle management
    startFuncs := []func(ctx context.Context) error{...}
    stopFuncs := []func(context.Context) error{...}
}
```

### Pain Points

1. **Excessive Initialization**: _Some_ Lambda functions create 20+ components when they need 3
2. **Manual Wiring Complexity**: UCAN server has 329 lines of dependency setup
3. **Configuration Explosion**: AWS service requires 45+ configuration fields
4. **Error Handling Fatigue**: Every initialization requires error checking
5. **Testing Nightmare**: Mocking aws.Construct requires recreating entire AWS stack
6. **Memory Overhead**: _Some_ Lambda cold starts suffer from unnecessary initialization

## Proposed Solution

Adopt Uber FX to provide:

1. **Automatic dependency injection**
2. **Lifecycle management**
3. **Clean separation of concerns**
4. **Enhanced testability**

### Example 1: FX-based AWS Service Module

Break the monolithic aws.Construct into focused modules:

```go
// aws/modules.go - Modular AWS service providers
var BlobStoreModule = fx.Module("blobstore",
    fx.Provide(
        ProvideS3Client,
        ProvideS3BlobStore,
        ProvideS3Presigner,
    ),
)

var AllocationModule = fx.Module("allocation",
    fx.Provide(
        ProvideDynamoClient,
        ProvideDynamoAllocationStore,
    ),
)

var PublisherModule = fx.Module("publisher",
    fx.Provide(
        ProvideIPNIStore,
        ProvideChunkLinksTable,
        ProvideMetadataTable,
        store.NewPublisherStore,
    ),
)

// aws/providers.go
func ProvideS3BlobStore(cfg *Config, client *s3.Client) *S3BlobStore {
    // No error handling needed - FX propagates errors
    return NewS3BlobStore(client, cfg.BlobStoreBucket, cfg.FormatKey)
}

func ProvideDynamoAllocationStore(cfg *Config, client *dynamodb.Client) *DynamoAllocationStore {
    return NewDynamoAllocationStore(client, cfg.AllocationsTableName)
}
```

### Example 2: Optimized Lambda Functions

Lambda functions only load what they need:

```go
// cmd/lambda/putblob/main.go
func main() {
    app := fx.New(
        // Only load blob-related modules
        fx.Provide(aws.LoadConfig),
        aws.BlobStoreModule,       // Just S3 blob store
        aws.AllocationModule,       // Just DynamoDB allocations
        
        // Handler only gets what it needs
        fx.Invoke(func(lc fx.Lifecycle, presigner Presigner, allocs AllocationStore, store BlobStore) {
            handler := blobs.NewBlobPutHandler(presigner, allocs, store)
            lambda.StartHTTPHandler(handler)
        }),
    )
    app.Run()
}

// Result: probably faster cold starts, and probably less memory usage.
```

### Example 3: Simplified UCAN Server

Replace 329 lines with clean module composition:

```go
// cmd/cli/serve/ucan.go
func startServer(cmd *cobra.Command, _ []string) error {
    app := fx.New(
        // Configuration
        fx.Provide(config.Load[config.UCANServer]),
        
        // Core modules
        datastores.Module,      // All datastore providers
        identity.Module,        // Identity and signer setup
        storage.Module,         // Storage service setup
        
        // Optional PDP integration
        fx.Decorate(func(cfg *config.UCANServer) *config.UCANServer {
            if cfg.PDPServerURL != "" {
                return fx.Provide(pdp.Module)
            }
            return nil
        }),
        
        // Start server with automatic lifecycle
        fx.Invoke(func(lc fx.Lifecycle, svc storage.Service, cfg *config.UCANServer) {
            lc.Append(fx.Hook{
                OnStart: func(ctx context.Context) error {
                    return server.ListenAndServe(
                        fmt.Sprintf("%s:%d", cfg.Host, cfg.Port),
                        svc,
                    )
                },
            })
        }),
    )
    return app.Start(cmd.Context())
}

// datastores/module.go - Reusable datastore module
var Module = fx.Module("datastores",
    fx.Provide(
        ProvideAllocationDatastore,
        ProvideClaimDatastore,
        ProvidePublisherDatastore,
        ProvideReceiptDatastore,
    ),
)

func ProvideAllocationDatastore(lc fx.Lifecycle, cfg *Config) (datastore.Batching, error) {
    ds, err := leveldb.NewDatastore(filepath.Join(cfg.DataDir, "allocation"), nil)
    if err != nil {
        return nil, err
    }
    
    lc.Append(fx.Hook{
        OnStop: func(context.Context) error { return ds.Close() },
    })
    
    return ds, nil
}
```
## Benefits

### 1. Lambda Optimization
**Current Problem**: Some lambdas initialize 20+ AWS services via `aws.Construct`
```go
// Current: aws.Construct creates:
// - S3 client + blob store
// - DynamoDB client + 4 tables (allocations, claims, chunks, metadata)
// - SQS clients (3 queues)
// - Multiple datastores
// - Publisher services
// - PDP integration
service, err := aws.Construct(cfg) // Creates everything
```

**FX Solution**: Lambdas only initialize what they need
```go
// putblob lambda only needs:
app := fx.New(
    aws.BlobStoreModule,    // Just S3 blob store
    aws.AllocationModule,   // Just allocations table
    // Skips: 18+ other services
)
```

**Expected improvements based on initialization overhead**:
- Fewer AWS SDK client initializations
- Reduced memory footprint from unused services
- Faster startup due to less initialization code

### 2. Code Reduction (Measurable)
**Current line counts**:
- `cmd/cli/serve/ucan.go`: 329 lines of initialization code
- `pkg/aws/service.go`: 434 lines for `Construct` and `FromEnv`
- `pkg/pdp/server.go`: 156 lines for `NewServer`

**Demonstrated in demo**:
- Traditional setup: 68 lines of main.go
- FX setup: 31 lines of main.go (54% reduction)
- Similar reductions expected for Piri based on demo results

### 3. Enhanced Testability
```go
// Test lambdas with minimal mocking
app := fxtest.New(t,
    fx.Provide(mockS3Client),
    aws.BlobStoreModule,
)

// Test UCAN server with mock datastores  
app := fxtest.New(t,
    fx.Replace(provideMockDatastores),
    storage.Module,
)
```

### 4. Modular AWS Services
```go
// Mix and match AWS modules as needed
var LambdaApp = fx.New(
    aws.ConfigModule,           // Always need config
    aws.BlobStoreModule,        // S3 operations
    aws.AllocationModule,       // DynamoDB allocations
    // Don't load: Publisher, Claims, Receipts, PDP, etc.
)
```

### 5. Feature Flags Without Code Changes
```go
// Conditional features based on config
fx.Decorate(func(cfg *Config) fx.Option {
    if cfg.EnablePDP {
        return fx.Options(pdp.Module)
    }
    return fx.Options() // No-op
})
```

## Risks and Mitigations

### Risk 1: Learning Curve
**Mitigation**:
- Comprehensive documentation and examples
- Gradual migration allowing time to learn
- Pair programming sessions

### Risk 2: Debugging Complexity
**Mitigation**:
- FX provides excellent debugging output
- Clear error messages for missing dependencies
- Dependency graph visualization tools

### Risk 3: Third-party Dependency
**Mitigation**:
- FX is mature and widely used (Uber, Cloudflare, etc.)
- MIT licensed
- Can be gradually removed if needed

## Alternatives Considered

1. **Google Wire**: Compile-time DI, but less flexible
2. **Manual Factories**: Current approach, proven problematic
3. **Custom DI**: High maintenance burden

## Conclusion

Adopting FX will modify Piri's architecture by:

### Immediate Benefits
1. **Reduced Lambda initialization** - Load only required services (2-3 vs 20+)
2. **Verified code reduction** - 54% fewer lines demonstrated in demo
3. **Modular AWS services** - Each lambda loads only what it needs
4. **Simplified testing** - Mock individual modules, not entire stacks

### Long-term Benefits
1. **Feature velocity** - Add new services without breaking changes
2. **Operational efficiency** - Reduced memory usage and costs
3. **Developer experience** - Clear module boundaries and dependencies
4. **Maintainability** - Automatic lifecycle management

### Measurable Improvements
**Code complexity** (verified in codebase):
- `aws.Construct`: Creates 20+ services when lambdas need 2-3
- UCAN server: 6 datastore initializations + complex option building
- Test setup: Current tests must mock entire service stack

**Expected improvements** (to be measured):
- Reduced initialization time from loading fewer services
- Lower memory usage from not holding unused service instances
- Faster test execution with targeted mocking

## References

- [Uber FX Documentation](https://uber-go.github.io/fx/)
- [demofx Proof of Concept Repository](https://github.com/frrist/demofx)
