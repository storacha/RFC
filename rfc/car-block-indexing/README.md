# The problem with using a CAR V2 Multihash Sorted Index as an external index

We store [CAR]s in buckets. Reads over http or bitswap have to read individual blocks from the CAR to build up the response.

To make a range request for just the bytes of a block we need to know the offset of the block bytes from the start of the CAR and it's byte length.

The [CAR V2 Multihash Sorted Index] is not designed to support this use case and does not give us that info.

In the terminology of CARs, a `Block` is a `(CID,bytes)` pair. When we handle a read request, we only really care about fetching the block `bytes`.

Each `Block` is also prefixed with a [varint] that specifies

> the number of remaining bytes for that block entry—excluding the bytes used to encode the integer, but including the CID for non-header blocks.

We currently store an Multihash Sorted Index for each CAR as an external file.

In the diagram below we want to know the offset and length of the block bytes sections, but all we are given is the offset of the `(len,cid,bytes)` tripple.

![carv2-multihash-sorted-index](https://hackmd.io/_uploads/HJy7pbr96.svg)

Parsing an index gives us a map from a block CID multihash to the offset of the block length varint.

The index **does not** give us the length of the block bytes nor the offset of the block bytes from the start of the CAR. To get the information we want we have to overfetch and guess.

The process today is

- fetch and parse the entire index for a CAR
- extract the multihash from the request CID
- read the offset for the Block from the index
- range request from CAR using that offset and assuming max block length ~ 1 MiB (!!!)
- read (and drop) the varint from the response bytes.
- read (and drop) the CID from response bytes.
- read the block bytes from the response bytes.
- drop the rest of the response bytes where we have overfetched.

Note: there is also so very detailed work in [freeway] that aims to expand the range requests to include batches of contiguous blocks where we know that we will need them... 

...but this still causes overfetching, and in resource constrained environments like Cloudflare Workers we have had to reduce the scope of that optimisation. The byte manipulation in memory caused resource exhuastion errors.

## Proposal

Let's make a CAR index format that makes it trivial to fetch block bytes from a CAR via range requests.

As a datapoint [hoverboard] currently uses our DynamoDB block index that stores `(multihash,carpath,offset,length)` for every block in every car, which allows us to make the range requests we want. 

But we also like having "index file per CAR" rather than 1 big db of all blocks. It allow us to move the index creation to the client, and index file per CAR preserves data locality ("many small indexes, each index records all blocks in that CAR")

A couple of encoding options seem reasonable.

### dag-cbor index

keep it simple: dag-cbor encode just the index data we need to make range requests, like an [inclusion content-claim] with the index inlined.

```js
{
  content: Link, // CAR CID
  includes: [
    { cid: Link /* block cid */, offset: number, length: number}
  ]
}
```

- `content` is the CAR CID that has been indexed, making this portable. the offsets declared are within that CAR.
- `includes` is an array of block index info, in the same order as they appear in the CAR.
  - `cid` is the Block CID
  - `offset` is the byte offset of the bytes of the block.
  - `length` is the byte lenth of the block bytes.


#### Encoded form

This format could be condensed using an IPLD schema that represents each item as a tuple rather than an object.

```ipldsch
type BlockIndex struct {
  cid: Link
  offset: Integer
  length: Integer
} representation tuple

type CarBlockIndex struct {
  content: Link
  includes: BlockIndex[]
}
```

The encoded index size for a 100MiB test CAR with 405 CIDv1 addressed ~256KiB blocks is:

| index format     | bytes  | diff |
|------------------|--------|------|
| multihash sorted | 16,230 | 0    |
| dag-cbor tuple   | 21,108 |+4,878 (+30%) |
| dag-cbor         | 28,398 |+12,168 (+74%) |


The encoded index size for a 114MiB test CAR from `w3cli` node_modules with 16,827 CIDv1 addressed blocks is:

| index format     | bytes   | diff | 
|------------------|---------|------|
| multihash sorted | 673,110 | 0
| dag-cbor tuple   | 838,294 | +165,184 (+24%)
| dag-cbor         | 1,141,180 | +468,070 (+70%)

The encoded index size for a 500MiB test CAR `web3-storage` node_modules with 68,051 CIDv1 addressed blocks is:

| index format     | bytes     | diff | 
|------------------|-----------|------|
| multihash sorted | 2,722,070 | 0
| dag-cbor tuple   | 3,387,570 | +665,500 (+24%)
| dag-cbor         | 4,612,488 | +1,890,418 (+69%)


the tuple represenation seems worth it, and dag-cbor encoding the CID+offset+length is still in the ball park of the multihash sorted index size.

### alt: cid sorted car v2 index

> This is documented to show my working. I'm not convinced anyone wants this to exists, but it follows the pattern existing pattern of CAR v2 indexes where IndexSorted lacked the multihash code, and so MultihashIndexSorted was born, and now we want the full CID... implying the existance of... `CIDIndexSorted`

An extention of the [CAR v2 Multihash Sorted Index], the CID Sorted Index records the CID  along with the offset and block bytes length.

By recording the CID and the block bytes length we can derive the header length and derive the bytes offset from the block offset + header length.

As a naive extrapolation from the multihash sorted index spec this could be achieved by prepending cid version and muticodec as additional bucket prefixes

```
| cidv (uint8?) | multicodec-code (uint64?) | multihash-code (uint64) | width (uint32) | count (uint64) | digest1 | digest1 offset (uint64) | digest1 byte length (uint64) | digest2 | digest2 offset (uint64) | digest2 byte length (uint64)...
```


[CAR]: https://ipld.io/specs/transport/car/carv1/
[varint]: https://en.wikipedia.org/wiki/LEB128
[CAR V2 Multihash Sorted Index]: https://ipld.io/specs/transport/car/carv2/#format-0x0401-multihashindexsorted
[inclusion content-claim]: https://github.com/web3-storage/content-claims?tab=readme-ov-file#inclusion-claim
[freeway]: https://github.com/web3-storage/freeway
[hoverboard]: https://github.com/web3-storage/hoverboard
