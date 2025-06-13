# RFC(Indexer): Multiple IPNI chains

## Background

The indexer publishes an IPNI advert chain with adverts for index and equality claims. For a short period yesterday the indexer was in production and receiving ALL equality claims for all uploads. It was observed to be behind the current head of the advert chain by multiple hours. It also reported a [`Lag`](https://github.com/ipni/go-libipni/blob/d8f9f451cc1d5d123f9ed270304211e9bc3678e6/find/model/provider_info.go#L19-L22) value of over 2,000 adverts.

The IPNI chain is a linked list and there are only so many opportunities to paralleise that work to increase ingest speed. Being behind the current head by such a large amount suggests that an IPNI node outside of the same datacenter/region as the source chain is unlikely to be able to keep up. That is to say, the IPNI node that was being observed was in the same region as the indexer - cid.contact is not.

## Proposal

The best way to increase ingest rate is to shard into multiple chains. At the very least we should shard into 2 chains - one for equality claims and one for index data.

Unfortunately aggregating multiple hashes into a single advert is not really possible since we use advert metadata - so all hashes in the advert have to carry the same metadata in order to be aggregateable. Note: we already do this for index data, but it is not possible for equality claims.
