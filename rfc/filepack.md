# RFC: Filepack archive format

**Author**: Alan Shaw (@alanshaw)

**Date**: 2025-09-05

**Status**: Draft

## Abstract

Filepack is an archive format for transferring content addressed data. It is comprised of two files: data archive and index. Both data and index MAY be split into multiple chunks.

The format is an alternative to CAR, allowing smaller and faster uploads as well as quicker retrievals.


## Format Definition

### Data Archive

A Filepack data archive is simply the concatenation of the bytes of 1 or more files and the bytes of 0 or more metadata items.

```
<file0-bytes>[...meta0-bytes][...fileN-bytes][...metaN-bytes]
```

For IPFS data, the metadata bytes may be the bytes of each non-leaf DAG node created in a UnixFS tree.

Filepack data MUST be identified as a whole by a CID, allowing it to be referenced by the filepack index. The IPLD codec used in the CID MUST be “raw” since the data is simply concatenated raw file bytes.

### Index

A Filepack index adds content addressing to the data by specifying hashes, and byte offsets of files within the data archive. It allows the data to be made sense of either partially or as a whole.

The index MUST specify the hashes of the data archives, the hashes of the files and the hashes of the metadata items.

The index MUST be organized such that if files have been split amongst multiple data archives, the file hashes are associated with the archive the file resides within.

The index MUST specify byte offsets (i.e. start + end) for each file and metadata item within the data archive. The implementation MAY alternatively specify offset and length.

The index MAY content address files in chunks i.e. specify multiple hashes for a given file that address different byte ranges.

A sharded DAG index can be used for this purpose.


## Rationale 

* More efficient retrieval even when file is split into multiple chunks - CAR format separates contiguous file bytes by a varint and a CID for each chunk. Yet file access from an IPFS Gateway is typically for the whole file, necessitating reassembly for any file bigger than the chunk size used when the data was imported. This can be dealt with in a few different ways but it always requires reassembly:
    * Over fetching (retrieving more bytes than necessary) involves fetching multiple blocks (chunks) from a CAR file in a single request and discards any varints and CIDs between blocks (wasteful and more expensive)
    * Fetching in multiple requests (potentially slower and at risk of exceeding rate limits)
    * Multipart byte range requests (increased complexity and processing)
* Conversely, retrieving a file from a Filepackd data archive consists of a single byte range request, regardless of the number of chunks the file may have been split into. No reassembly occurs, just streaming the bytes.
* In the case where a data archive consists of a single file, an index is not necessarily needed since the hash of the archive may be hash of the file (assuming no metadata). This makes content discovery and retrieval much faster.
* Saves space by shedding the overhead that the CAR file format adds.
* Storing data as flat file bytes allows content addressing to be layered on top, allowing the mechanism for content addressing to change without changing the bytes of the data. Files can be content addressed IPFS style, where each file is chunked into multiple content addressed blocks. Alternatively the file can be content addressed as a whole, or via some other hashing algorithm that allows incremental verification.
* Backwards compatible with existing Storacha storage and retrieval patterns.
