# RFC(Indexer): Do not use cache when publishing claims

## Background

The indexer is the publishing target for two claim types - "assert/index" and "assert/equals". Both of which are written to an IPNI advert, long term storage _and_ the cache. There is special handling for index claims, where every slice in the index is also cached.

IPNI advert chains are crawled at unknown rate, meaning that there is an unknown amount of time between the indexer writing an advert and it's contents being available to query from an IPNI node. Caching these claims allows read on write semantics for data uploaded to the network in the period between the advert being written and it being ingested by an IPNI node.

However, caches are typically ephemeral in nature. We use a LFU cache which means that items may be pushed out of the cache _before_ an IPNI node ingests the advert that contains the cached data.

To mitigate this we use a "volatile" LFU cache and add items with _no expiry_, meaning they cannot be evicted from the cache, ever. We have a separate process that monitors an external IPNI node's progress ingesting the advert chain and expires items as it progresses.

Unfortunately this can create an undesired situation when a lot of claims are published in a short space of time (or when an index claim contains an index for a large number of hashes) - Out of Memory (OOM).

This prevents the indexer from successfully accepting claim publications and also prevents it from servicing queries due to cache write on read (see [#57](https://github.com/storacha/RFC/pull/57)).

The indexer also receives location commitments ("assert/location") from storage nodes via a "claim/cache" invocation that are critical to an upload flow completing. Currently these are not cached with _no-expiry_ and as such it is simply chance that allows uploads to complete without failing.

Finally, indexes for especially large DAGs can be used as an attack vector - they can cause DoS by using the entirely of the cache memory for unexpireable items and they can have an adverse affect on performance by eclipsing a large portion of the cache. It was recently observed that the current indexer cache was full at around 18,000 keys - which roughly equates to a single 10k NFT drop (10k blocks for assets and 10k blocks for metadata).

## Proposal

Temporal locality would suggest that we need to cache everything on upload, but I propose that as DAGs size increases, the propably of access becomes less likely for the _entirety_ of the DAG, although granted probably still as likely for _random parts_ of the DAG and/or for nodes encountered first in depth first traversal. Meaning that for a big DAG we just don't know what to cache and we should just "cache" everything.

Hence, the proposal is to not cache this data in the same cache written to by a query. Also, do not cache this data in a size bounded cache, since all the data we're describing MUST persist until an IPNI node has ingested our advert chain.

A key-value store or a regular database would serve this purpose. It will not respond as quickly as an in-memory cache but would mitigate all issues outlined above.

We might be able to avoid doing the work (and cost) of removing from the data store by chosing one that supports TTL, and simply setting it to a large enough value such that we are confident an IPNI chain will have ingested an advert by the time the item expires. Location commitments likely need either a very long TTL like this or special handling to remove from the cache when an IPNI node is observed to respond to a query for them.

The consequence is that the cache for publishing is no longer a fixed cost, and will scale with the rate of uploads. However, it is necessary to allow uploads to succeed without being affected by other uploads in the system, and is necessary to allow the rate of uploads to scale beyond the capacity of the cache used for queries.
