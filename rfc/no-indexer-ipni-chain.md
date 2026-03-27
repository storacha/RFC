# No IPNI chain on the indexing service

In a conversation, Vicente asked why the indexer must also maintain a chain and why the index data for a blob is not stored on the storage node, in its own IPNI chain.

There a few reasons for this, please comment if I have missed some:

1. Read on write - the storage node is not the indexing service, if you send it the index claim you'll have to wait for that info to get into IPNI before you can query the indexing service for it.
1. Separation of concerns - indexing is not storage.
1. ...

## Proposal

1. Move the IPNI index data to the storage node.
1. Send `assert/index` invocations from the upload service to the storage node that received the blob that is being indexed.
1. Storage node adds each multihash to an IPNI advert on the storage node in the same way the indexing service does right now.
1. Have the storage node send an onward `claim/cache` to the indexing service as part of the invocation so we can retain read-on-write semantics.

## Pros/cons

* ✅ There's incentive for the storage node to maintain the chain - since it allows the data to be fetched and them to get paid egress.
* ✅ It allows the indexer to be ephemeral - a true cache of data.
* ✅ Removing the index when removing a blob is simplier.
* ❌ If data is moved, we need to re-index.
* ❌ Trusting the storage node to store the index claim in addition to the location claim seems risky from a data availability point of view.
* ❌ IPNI chain data for the index is not currently part of the replication protocol, we'd need to extend to allow the index to be replicated in addition to the blob `space/index/replicate`?
* ...

## Discussion

Why not also the equivalency claim?
