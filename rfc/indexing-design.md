# W3 Indexing Design
> Replaces [W3 IPNI Indexing RFC](https://github.com/w3s-project/RFC/blob/main/rfc/ipni-w3c.md)

## Authors

- [gammazero]
- [gozala]

## Goals

A user can use a multihash, that identifies data stored on w3s, to lookup the all information necessary to retrieve and delegate access to that data.

Location commitments that describe where content is stored can change over time as w3s moves data to a more favorable location, re-aggregates data storage, or replicates stored data. Location commitments are updated without having to index all multihashes.

A query API will allow a user to specify indexed content to lookup and how to filter results. The API retrieves previously cached responses or composes new IPNI queries to get new responses to filter and cache.

## Overview

A user stores content in shards of opaque blobs that contain a series of blocks addressed by multihashes. Blobs are set of blocks in CAR file format (although they could use different format in the future) and are stored at a location designated for a particular user's data.

The agent uploading user content builds a UnixFS IPLD DAG and encodes it into one or more blobs. It also derives ([sharded-dag-index](https://github.com/w3s-project/specs/blob/main/w3-index.md#sharded-dag-index-example)) that describe blob in terms of (multihash, offset, length) tuples for all the blocks. This enables the user to make choices about which block to be make discoverable and retrievable over the network. For example, they may want only the DAG root to be indexed.

The blob-index is stored in the user's space along with the CAR file(s) containing the blob data. This is done in the [w3 add-blob operation](https://github.com/w3s-project/specs/blob/main/w3-blob.md#add-blob). The w3s service creates [location commitments](https://github.com/w3s-project/specs/blob/main/w3-blob.md#location-commitment) for the blob shards and for the blob-index. 

After the add-blob process completes, and the user receives an [accept-blob receipt](https://github.com/w3s-project/specs/blob/main/w3-blob.md#accept-blob-receipt), the user may then choose to make their data publicly queryable by publishing it to W3UP's indexing system.  The user optionally invokes an [index-add](https://github.com/w3s-project/specs/blob/main/w3-index.md#index-add) capability to publish the blob-index multihashes into W3UP's indexing system and eventually to [IPNI](https://github.com/ipni/specs/blob/main/IPNI.md) so that they can be used to look up location commitments and retrieve blob-index information and blob data. See [W3 Index](https://github.com/w3s-project/specs/blob/main/w3-index.md#w3-index) for more.

After publishing to IPNI, any user looking for content can query IPNI for the blob-index of that content. The user read the blob-index to determine what data shard needs to be retrieved, and then asks IPNI for the location of that shard.

Finally the user uses the shard location, with range information from the blob index, to retrieve a specific piece of data from w3s.

## Index Query

A w3s client can query indexed information by specific attributes, and can filter retrieved query results.

The queryable attributes are as follows.
- Site: This lets the query ask for results where data is stored at specified site(s), i.e. query can specify "where site in [X]".
- Shard: This lets the query ask for results for a specific shard, i.e. query can specify "where shard in [S]".
- Slice: This lets the query ask for a specific block of data, identified my multihash with a result being the byte range within a shard containing the block of data.

Query results can be filtered to select only results that the user has access to, or for data supplied by specific data providers.

### Query Specification

```
match = {
  subject=[myID, myOrgID],
  location=spBaseURL,
}
results = Query(multihash="bafy...", match)
```

### Query Operation Diagram

![w3up-query-diagram](w3up-query-diagram.png)

## Indexing Components

The indexing components are intended for deployment on the w3up gateway. They may also be deployed to the w3up client, where they will not be shared with other clients.

### W3up Index

The w3up index layer is responsible for reading a query specification, and then
1. Creating a set of IPNI queries needed to get the requested index content
2. Sending the queries to the w3up IPNI cache
3. Filtering returned results
4. Packaging results for the query client.

### W3up IPNI Cache

The w3up IPNI Cache is a cache that holds IPNI query results. The cache is:

- Limited-size LRU. Discards the least recently used data when at storage capacity.
- Temporary. Evicts items that have been cached for longer than the cache expiration time, regardless of last access. This allows changes to query results to be seen.
- Negative (empty) result-aware. Caching empty results prevents more expensive IPNI queries for data that is not indexed. Negative cache entries are kept in a separate cache so that negative entries cannot evict a positive entrries due to LRU.
- Populated on write: When new or updated data is publisher to IPNI, the cache is populated. This replaces previous any previous cache entries, including negative ones.

This is a read-through cache, meaning if it does not hold the request query results, then it forwards the query on to IPNI, and caches the results. This includes caching an empty response.

There are two types of queries send to the cache. Each takes a data multihash, but returns a different result:
1. shard/slice query. Returns one or more: shard CID, slice CID, offset, length (in shard).
2. location query. Returns: location commitment of each shard. 

By using both these query results, a specific portion of the blob data can be retrieved. The shard(s) are read from the locations in the location commitments, and data ranges within each shard are requested (when not reading an entire shard) by range requests using the offsets and lengths from the shard/slice data.

If either shard or location results are not cached, then a query for the multihash is passed to the IPNI client.

#### Expiration Times
The shard/slice data does not change over the lifetime of the stored data, so this data can have a much longer expiration time. It could also have no expiration time and be explicitly removed from cache when the data is no longer stored, but this means that there need to be a communication path from data removal to all cache locations. So, better to just have an expiration time, making cache more independent.

### IPNI

#### Get w3up results

IPNI maps a multihash to location and metadata information. When queried with a multihash, IPNI will respond with all known locations/metadatas of that multihash. The w3up IPNI query client must filter these results to only select results that are under the control of w3up.

Example IPNI query and response:
```bash
curl https://cid.contact/cid/bafybeicawc3qwtlecld6lmtvsndimoz3446xyaprgsxvhd3aapwa2twnc4 -sS | jq
```
```json
{
  "MultihashResults": [
    {
      "Multihash": "EiBAsLcLTWQSx+WydZNGhjs75z18AfE0r1OPYAPsDU7NFw==",
      "ProviderResults": [
        {
          "ContextID": "YmFndXFlZXJheTJ2ZWJsZGNhY2JjM3Z0em94bXBvM2NiYmFsNzV3d3R0aHRyamhuaDdvN2o2c2J0d2xmcQ==",
          "Metadata": "gBI=",
          "Provider": {
            "ID": "QmQzqxhK82kAmKvARFZSkUVS6fo9sySaiogAnx5EnZ6ZmC",
            "Addrs": [
              "/dns4/elastic.dag.house/tcp/443/wss"
            ]
          }
        },
        {
          "ContextID": "YmFndXFlZXJheTJ2ZWJsZGNhY2JjM3Z0em94bXBvM2NiYmFsNzV3d3R0aHRyamhuaDdvN2o2c2J0d2xmcQ==",
          "Metadata": "oBIA",
          "Provider": {
            "ID": "QmUA9D3H7HeCYsirB3KmPSvZh3dNXMZas6Lwgr4fv1HTTp",
            "Addrs": [
              "/dns4/dag.w3s.link/tcp/443/https"
            ]
          }
        }
]}]}
```

#### Get sharded-dag-index

After selecting only w3up results, these must then be processed to generate responses for shard/slice and location query attributes. To do this, first the blob-index result is read from the IPNI result metadata. This gets the CID of the sharded-dag-index (aka blob-index) and the location portion of the result gives the base URL to retrieve the blob-index from.

The blob-index is retrieved and read. The blob-index data is searched to find the multihash, and to get the shard and slice (if the multihash is a slice within a shard).

#### Get locations of shards

After getting the shard and slice data, then location commitment data needs to be collected.

If the multihash is that of a whole shard, then the location commitment results are already in the IPNI results. If the multihash is a slice, then another IPNI query needs to be made to retrieve the location(s) of the shard, using the parent shard CID from the blob-index.

The location commitments are read from IPNI metadata and are returned to the cache.

## IPNI Advertisement
Information is provided to IPNI in the form of [IPNI Advertisements](https://github.com/ipni/specs/blob/main/IPNI.md#advertisements) that are signed by w3s and published to IPNI by w3s. When a new advertisement is published, an [IPNI Announcement](https://github.com/ipni/specs/blob/main/IPNI.md#announcements) is sent to IPNI to trigger IPNI to fetch the new advertisement.

A separate advertisement chain is used for location commitments and blob-index data.

The reason for this is that location information is subject to change, but blob index data will not change over the lifetime of the stored data.

### Advertisement Structure

An IPNI Advertisement has the IPLD Schema:

```ipldsch
type Advertisement struct {
    PreviousID optional Link
    Provider String
    Addresses [String]
    Signature Bytes
    Entries Link
    ContextID Bytes
    Metadata Bytes
    IsRm Bool
    ExtendedProvider optional ExtendedProvider
}

type EntryChunk struct {
    Entries [Bytes]
    Next optional Link
}
```

### IPNI Advertisement for sharded-dag-index

Map all multihashes to blob-index CID/location
ContextID = hash(userKey + blob-index CID)

- `ProviderID` libp2p peer ID of w3s blob-index publisher.
- `Addresses` the most recent value of this is returned with lookups for any CID in any advertisement. It will contain the multiaddr form of the blob-index publisher URL. This is where the blob-index is retrieved from when presented with the blob-index CID.
- `Signature` Signature over all advertisement data. Created using private key matching `ProviderID`.
- `ContextID` byte encoded userKey + blob-index CID.
- `Entries` must be the CID of an `EntryChunk` for a subset (or all) of the multihashes in the blob-index.
- `Metadata` encodes the CID of the blob-index + version info is any.
- `IsRm` is used when removing all advertisement(s) that have that context ID and deleting all associated multihash indexes from IPNI.

The set of multihashes within one blob must be encoded as 1 Advertisement. If a blob is partitioned into multiple sub-blobs this still generates one advertisement as there is still one blob index.

Each advertisement encodes the multihashes into 1 or more EntryChunk blocks. EntryChunk blocks are an array of multihashes with a link to a next block, if there is another block. The EntryChunk serves a similar purpose as HTTP chunking. 

All multihashes must be sorted in ascending order before being split into chunks. This is done to have deterministic output when creating entry chinks - the same set of multihashes should generate the same set of entry chunks.

### IPNI Advertisement for location commitment

Map shard-multihash to shard CID/location
ContextID = hash(userKey + shard CID)

- `ProviderID` libp2p peer ID of w3s location publisher.
- `Addresses` multiaddr of location commitment URL. Only used if location commitment does not fit into metadata.
- `ContextID` byte encoded userKey + shard CID
- `Entries` must be the CID of an `EntryChunk` that contains shard CID.
- `Metadata` encodes the shard's location commitment.
- `IsRm` is used when removing the shard from storage

The reason that the shard ID is used to construct the context ID is because individual shards may change location, so only the index information for that shard needs to change.

NOTE: Consider putting the non URL portion of the location commitment into the `Addresses` field and the rest in the metadata.

## Blob Deletion

Removal advertisements are placed on both the blob-index chain and the location chain to delete the blob-index and shard(s) for the user.

To get the shard CID(s), then blob-index must first be read. This may be read from cache as it does not change. The shard CID are read from the blob-index.

## Dependable Reads

Dependable reads are required for ensuring that updates have completed. This means that following a write of index data, that that data can reliably be read.  This makes it necessary to have a synchronous write/update operation, after which reads will return the data that was written.

This synchronous write path can be something supported by IPNI. However, that would require a non-standard indexer feature or extension of the protocol. This feature would not be something that would be generally desirable for a global indexer, and may only be appropriate for a site-local indexer.

Another way to support a synchronous write path is to write to the IPNI cache on write, and guarantee that the data remained in the cache until it is indexed in IPNI. An advantage of this write path is that it does not rely on any non-standard behavior of IPNI. It is also more reliable since the critical happens on w3s infrastructure. This ensures that we control the time needed for a read to succeed.

### Cache on write

When an advertisement is created to publish new index data, the IPNI cache is populated with the data retrievable by index queries. An attribute is set on this data that prevents it from being evicted from the cache until it has been verified to be indexed. This ensures that after cache eviction, lookup of the now uncached data will succeed.

## Design questions

1. What are the trade-offs of using IPNI to lookup location commitments as opposed to bundling these with blob-index data.
2. Deploy on w3s gateway, client, both?
3. How can the IPNI cache be distributed? Should its contents be shared on IPFS?
