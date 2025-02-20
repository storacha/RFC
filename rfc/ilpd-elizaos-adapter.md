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

### IPLD Schema

The idea is define the IPLD schemas for the following data structures:

1. **MemoryBlock**: Represents individual memory entries (messages in the agent chat) with metadata and content links.
2. **CollectionBlock**: Represents collections of memory blocks with chronological ordering.
3. **RootBlock**: Represents the root index mapping collection names to their index blocks.

Later this can be expanded to other data types we may want to store.

### IPLD Helper Class

An `IPLDHelper` class will be implemented to handle encoding and decoding of IPLD blocks, as well as creating CAR (Content Addressable aRchive) files for storage.

### Integration with Storacha

The `StorachaDatabaseAdapter` will be modified to:

- Encode memory data into IPLD blocks before storing it into Storacha.
- Use IPLD's content addressing for data retrieval.
- Update the index with new CIDs for stored data.

## Drawbacks

- Increased complexity in data handling due to the introduction of IPLD.
- Potential learning curve for developers unfamiliar with IPLD concepts.
- Write & Read performance might not be similar to other database adapters.

## Alternatives

- Continue using the existing JSON-based storage system. Not very efficient.
- Explore other content-addressable storage solutions.

## Unresolved Questions

- How will IPLD adoption impact performance in agent with a large memory history?

## Implementation Plan

1. Define IPLD schemas for memory storage (we don't need to define the schema for all the data types for now).
2. Implement the `IPLDHelper` class for encoding/decoding and CAR file creation.
3. Modify the `StorachaDatabaseAdapter` to integrate IPLD.
4. Update documentation.

It MUST be transparent to the client that calls the Storacha Database Adapter.

## Conclusion

Adopting IPLD for memory storage in the Storacha Database Adapter offers significant benefits, including enhanced data integrity, interoperability, scalability, and seamless data sharing. However, it is important to recognize that write operations may not achieve the same efficiency as inserts and updates in traditional relational databases - which are commonly used as database adapters. This trade-off is a common consideration in systems that prioritize distributed storage and data verifiability.