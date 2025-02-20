# RFC: Adoption of IPLD for Memory Storage in Storacha Database Adapter (ElizaOS)

## Authors

- [Felipe Forbeck](https://github.com/fforbeck), [Storacha Network](https://storacha.network/)

## Introduction

This RFC proposes the adoption of IPLD (InterPlanetary Linked Data) for storing and managing data within the Storacha database adapter for ElizaOS. The goal is to leverage IPLD's content-addressable and verifiable data structures to enhance data integrity, interoperability, scalability, and data sharing capabilities.

## Motivation

The current memory storage system in Storacha relies on traditional JSON-based storage, which lacks the benefits of content addressing and verifiability. By adopting IPLD, we aim to:

- Enable seamless data sharing between different agents and systems.
- Improve data integrity through content-addressable storage.
- Enable efficient data retrieval and sharing across distributed systems.
- Facilitate interoperability with other IPLD-compatible systems.

## Detailed Design

### Index/Manifest System

The index/manifest system efficiently manages collections of data stored in IPFS:

- **Root Index**: Maps collection names to their CIDs (Content Identifiers) in IPFS, acting as a directory for locating collection indexes.
- **Collection Index**: Contains metadata for each item in a collection, including CIDs, filenames, and timestamps. It ensures data integrity and order through sequence numbers and root CIDs.

Data retrieval involves querying the collection index for CIDs and using them to fetch data from IPFS, ensuring content-addressable storage and efficient data management.

### Data Sharing

IPLD's content-addressable nature allows for easy data sharing across different agents and systems. By using CIDs, data can be referenced and accessed globally without duplication. This enables:

- **Shared History**: Agents can share a common root index CID, allowing them to access and update shared collections seamlessly.
- **Versioning and Provenance**: IPLD's immutable data structures provide a clear history of changes, enabling version control and provenance tracking.

Example of data-sharing in ElizaOS: https://github.com/storacha/elizaos-adapter/blob/main/docs/integration.md#sharing-between-agents

### IPLD Schema

The idea is to define IPLD schemas for the following data structures:

1. **MemoryBlock**: Represents individual memory entries (messages in the agent chat) with metadata and content links.
    ```javascript
    import { schema } from '@ucanto/core';

    const MemoryBlockSchema = schema({
      id: 'string',
      content: 'Link',
      roomId: 'string?',
      tableName: 'string?',
      agentId: 'string?',
      created: 'string',
      updated: 'string',
      sequence: 'number?',
      embedding: 'array?',
      previousBlock: 'Link?',
    });
    ```
2. **CollectionBlock**: Represents collections of memory blocks with chronological ordering.
    ```javascript
    import { Link } from 'multiformats';

    export type CollectionBlock = {
        items: Link[]; // Links to memory blocks
        lastUpdated: string; // ISO date string
        lastSequence: number; // Last used sequence number to track the order of the memories
        rootBlock?: Link; // Link to the first block in the collection
    };
    ```
3. **RootBlock**: Represents the root index mapping collection names to their index blocks.
    ```javascript
    import { Link } from 'multiformats';

    export type RootBlock = {
        collections: { [name: string]: Link }; // Map collection names to their index blocks
        lastUpdated: string; // ISO date string
    };
    ```
These schemas can be expanded to other data types as needed.

### IPLD Helper Class

An `IPLDHelper` class will be implemented to handle encoding and decoding of IPLD blocks, as well as creating CAR (Content Addressable aRchive) files for storage.

```javascript
import { CID } from 'multiformats';
import * as Block from 'multiformats/block';
import * as codec from '@ipld/dag-cbor';
import { sha256 as hasher } from 'multiformats/hashes/sha2';
import { CarWriter } from '@ipld/car';
import { Blob } from '@web3-storage/w3up-client';
import type { MemoryBlock, CollectionBlock, RootBlock } from './schema.js';

export class IPLDHelper {
    /**
     * Creates a CAR file containing a single block
     * @param value - The value to encode and store in the CAR file
     * @returns A Blob representing the CAR file
     */
    static async createCAR(value: any): Promise<Blob> {};

    /**
     * Creates a memory block in IPLD format
     * @param memory - The memory object to encode
     * @param contentCID - The CID of the content linked to this memory
     * @returns An object containing the encoded memory block and its CID
     */
    static async createMemoryBlock(memory: any, contentCID: CID): Promise<{ block: MemoryBlock; cid: CID }> {};

    /**
     * Creates a collection block in IPLD format
     * @param items - An array of CIDs representing the items in the collection
     * @returns An object containing the encoded collection block and its CID
     */
    static async createCollectionBlock(items: CID[]): Promise<{ block: CollectionBlock; cid: CID }> {};

    /**
     * Creates a root block in IPLD format
     * @param collections - A map of collection names to their index block CIDs
     * @returns An object containing the encoded root block and its CID
     */
    static async createRootBlock(collections: { [name: string]: CID }): Promise<{ block: RootBlock; cid: CID }> {};
}
```

### Integration with Storacha

The `StorachaDatabaseAdapter` will be modified to:

- Encode memory data into IPLD blocks before storing it into Storacha.
- Use IPLD's content addressing for data retrieval.
- Update the index with new CIDs for stored data.

## Drawbacks

- Increased complexity in data handling due to the introduction of IPLD.
- Potential learning curve for developers unfamiliar with IPLD concepts.
- Write and read performance might not match that of other database adapters.

## Alternatives

- Continue using the existing JSON-based storage system, though it is less efficient.
- Explore other content-addressable storage solutions.

## Unresolved Questions

- How will IPLD adoption impact performance in agents with a large memory history?

## Implementation Plan

1. Define IPLD schemas for memory storage (we don't need to define the schema for all data types initially).
2. Implement the `IPLDHelper` class for encoding/decoding and CAR file creation.
3. Modify the `StorachaDatabaseAdapter` to integrate IPLD.
4. Update documentation.

The integration should be transparent to clients using the Storacha Database Adapter.

## Conclusion

Adopting IPLD for memory storage in the Storacha Database Adapter offers significant benefits, including enhanced data integrity, interoperability, scalability, and seamless data sharing. However, it is important to recognize that write operations may not achieve the same efficiency as inserts and updates in traditional relational databases, which are commonly used as database adapters.


### Additional Resources

Docs

- https://elizaos.github.io/eliza/docs/core/agents/#overview

- https://elizaos.github.io/eliza/docs/core/database/


Notes

- https://github.com/storacha/elizaos-adapter/blob/main/docs%2Fintegration.md