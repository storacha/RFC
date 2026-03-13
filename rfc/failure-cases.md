# RFC: What to do in the failure cases?

Ideas are here needed.

The Storacha Network is not setup well to deal with failure cases. We have implemented the happy path, but there are various things that can go wrong that we currently have no story for. This RFC attempts to enumerate a bunch of them that currently live in my head, with the hope that we can gain consensus on what _should_ happen and allow further RFC(s) to be opened and/or specs created/amended to deal with them.

* Replication failure
    * What happens when a replication does not succeed? Currently nothing.
* Data deletion
    * Long standing problem. We have a better story for this on storage nodes but no implementation. An interesting/difficult problem is how to deal with deletion in the context of our PDP root aggregates as well as our Filecoin deal aggregations. There is also data stored on the IPNI chain on the indexer that needs to be cleaned up. Or consider implementing [RFC #52](https://github.com/storacha/RFC/pull/52)
* Data loss
    * If a node loses some data, what happens? How do we know? We should probably be less inclined to use them for storing data. How do we ensure minimum replicas are maintained? This is linked to proving failure.
* Proving failure
    * If a node falls below the successful proving threshold, what happens? We probably should NOT attempt to store _more_ data on that node.
* Allocation failure
    * If we cannot allocate on a node, we should try a _different_ node. We should consider that an allocation failure could be temporary and caused by a full disk or adverse network conditions. Aside - we should consider increasing probability of allocating on nodes that are close to the client uploading the data to mitigate firewalls (inability to reach the node) and allow for faster upload times.
* Accept failure
    * After uploading a blob the upload service should at least retry a blob/accept invocation to a storage node. If the node responds with an error receipt, what is the implication? Has the node failed to store the data? How should it affect our desire to store _more_ data on the node?
* Retrieval failure
    * How do we observe and verify failures to retrieve? How do we prevent storage nodes from accepting retrieval invocations, claiming the egress and not sending the data?
* Node leaving the network
    * How do we detect this and clean up? How do we repair all the data it was storing? We should stop considering that node as a candidate for uploads.

A lot of these failure cases can boil down to us maintaining node weights, and how each one of these situations affects a weighting. That said, it would be good to find some time to consider whether weights is the right solution or if there is a better alternative, like a broader reputation system, that is maybe provable and observable. There is some much harder problems here though and it would be good to find some time to consider what we could put in place to either solve or align incentives in a way that makes it undesirable.
