# RFC: Virtual DAG in Sharded Dag Index

**Author**: Hannah Howard (@hannahhoward)

**Date**: 2025-09-07

**Status**: Draft

## Abstract

This RFC proposes augmenting the sharded dag index format to store metadata blocks directly in a sharded DAG Index, instead of one of the underlying shards. In combination with the Filepack data archive format (./filepack.md) this could enable valuable properties -- such as the ability to store system files up to the shard size limit in their original format.

## Motivation

UnixFS is an essential platform for representing files and folders in ways that are easy to transmit in reasonable <2MB size chunks. We require it for IPFS compatibility and for storing directories generally.

At the same time, it's a complex specification and not really web compatible, which has led to popular simpler alternatives like DASL (https://dasl.ing/).

We've already encountered the problem of storing larger than 1MB files by their raw link in Bluesky backups. We end up CARifying the post attachments, which Bluesky references by a single sha256 raw CID for the whole attachment. Then we have to store metadata in the bluesky backup app itself that connects the raw sha256 to the associated UnixFS DAG we generate during upload.

## Format Definition

We propose to augment the Sharded DAG Index as follows:

```ipldsch
type ShardedDagIndex union {
  | ShardedDagIndex_0_1 "index/sharded/dag@0.1"
  | ShardedDagIndex_0_2 "index/sharded/dag@0.2"
} representation keyed

type ShardedDagIndex_0_1 struct {
  content Link
  shards [Link]
} representation map

type ShardedDagIndex_0_2 struct {
  content Link
  shards [Link]
  blocks [Link]
} representation map

type Multihash bytes

type Position struct {
  offset Int
  length Int
} representation tuple

type BlobSlice struct {
  multihash Multihash
  position Position
} representation tuple

type BlobIndex struct {
  multihash Multihash
  slices [BlobSlice]
} representation tuple

```

The `blocks` attribute in a sharded DAG index v0.2 block is simply a collection of links to blocks that are included in this index file. Since a Sharded DAG Index is a CAR file, these blocks are simply inserted into the CAR file.

**IMPORTANT**: While small, this represents a breaking change for retrieval clients. Because the blocks included in the index are no longer in the underlying shard file, a retrieval client MUST be able to read the additional blocks out of the Sharded DAG Index CAR file in order to perform the retrieval. This is why we change the version of the Sharded DAG Index and list the included blocks in the Sharded DAG Index v0.2 root block.

## Benefits

Adding the ability to put blocks directly into the Sharded DAG Index would enable us to store files as blobs in their original format (up to the shard size), while maintaining full UnixFS compatibility.

While a Sharded DAG Index would be used to fetch the UnixFS representation of a file for a transport like bitswap, the Blob CID would also be a DASL compatible direct link to the raw file.

This would provide much faster RTT when using w3s.link -- the block could be returned directly from the location claim, and with no range request. Because the blob hash is sha256, it could be verified directly by the browser's various data integrity tools.

This approach could enable other optimizations as well. For many UnixFS directories, the entire directory structure could live in the sharded DAG Index. This would enable deep pathing in only two roundtrips -- one to fetch the sharded dag index, and one to fetch the underlying file.

## Alternative Approach

Another alternative to consider: split the metadata blocks into a separate blob, continue to use filepack and v0.1 sharded dag indexes. This would allow full compatibility, if we could allow metadata-only filepacks.

## Considerations

We still need to accommodate large files, and we need to consider whether we'd want to upload a directory of small files as a sharded dag index plus a bunch of small blobs or a single filepack blob. Additionally, while metadata blocks are generally quite small, we could still end up with large sharded dag indexes if we aren't careful.