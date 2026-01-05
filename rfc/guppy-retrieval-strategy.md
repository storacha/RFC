# Guppy Retrieval Strategy

## Multi-block requests

When we retrieve data, how should we make requests to the storage nodes? We want to minimize the overhead of many requests, but also minimize the overhead of over-fetching data. We have a few options:

* **Naive:** On every request for a block, look up its location, and fetch exactly that block.
  * Each block is retrieved in a separate request.

* **Shard-Optimistic:** When the first block from a shard is requested, fetch the entire shard blob and cache it.
  * Only one request is made per involved shard, but we may fetch (and egress) more data than required.
  * We also need to hold onto the cached shards until we're done using them, and those shards potentially take up much more space than the actual target data.

* **Range-Coalescing:** When a block is requested, place it in a queue. Periodically, for each shard with blocks in the queue, coalesce the ranges of those blocks, and then request each range. The shards are (currently) CAR files, so adjacent blocks are not literally adjacent (there's a CID and length between them), so we count blocks that are "close enough" as adjacent.
  * If the requested data involves many continguous blocks, this involves many fewer requests than the Naive approach, but not as few as the Shard-Optimistic approach.
  * Like the Naive approach, it retrieves the minimal blocks, though unlike the Naive approach it must egress the CID and length data between blocks, which is then ignored.
  * Startup is slow, because only the root block can be fetched on the first request. Efficiency on further rounds is best on wide DAGs and worse on deep DAGs.
  * This approach can also be tuned to coalesce across larger gaps, incurring some of the benefits and costs of the Shard-Optimistic approach when blocks are nearby each other but not directly adjacent.
  * There is also a multipart version of this approach which can make a single request for multiple ranges at once, but server support for this is spotty, and notably lacking in Go. It also incurs overhead in the response which may negate any benefits.
  * Currently, Freeway implements a Range-Coalescing approach, so we have some evidence it works decently.

* **Chunk-Optimistic:** This is the same as Shard-Optimistic, but divides shards into smaller chunks. When the first block is requested, a range of nearby blocks are fetched along with it.
  * Like Range-Coalescing, this strikes a balance between Naive and Shard-Optimistic.
  * Unlike Range-Coalescing, startup can include multiple blocks.
  * Unlike other approaches, the ranges may not be on block borders, because the borders of blocks are unknown until we look up the block in the index. That may make managing cached data difficult to manage.

### Thoughts

* For large data, the startup cost of Range-Coalescing is much less significant. Large data is also (warning: speculation) more likely to be wide than deep. The only way to make an especially deep UnixFS DAG would be to start with a very deep directory tree.

* Range-Coalescing takes some effort, but it's a strategy we've implemented before, in JS, and it seems to provide a good balance.

* Chunk-Optimistic is probably not very useful.

* Metrics are probably the key to tuning.

* Any time we egress data that's ultimately discarded, we should have a pretty strong argument for why, since egress is charged to the customer.

## Managing HTTP requests

Things we can do:

* Open a connection to each available storage node and pool them.
* Round-robin requests.
* Make sure nodes support HTTP/2 and use its pipelining. *(I think we already do?)*

## Possible Metrics

Metrics that might indicate whether our strategy is serving us, and if not, what to address:

* Egressed bytes desired vs overhead
* Number of fetches, and bytes / fetch
* Total retrieval byte rate
* Retrieval byte rate per connection
* Inactive time per connection
* Request overhead (if we can measure this)

## Current proposal

* Range-Coalescing as implemented in Freeway.
* All of the connection management ideas above.
