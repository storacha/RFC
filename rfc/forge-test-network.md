# Forge test network

This RFC proposes a stable test network for Forge that can be used as a bootstrapping/onboarding playground for node operators and customers. It also provides Storacha with a close-to-production area to sanity check network upgrades and a place to vet node providers before they are promoted to production.

The Forge network requires customers to commit to storage capacity up front and node providers to run a solid production setup. Both of these are significant hurdles to using the production network. The test network aims to lower that boundary by providing a place where potential customers can try out the network - allowing experimentation to take place and failures to happen without financial consequence. In short, the test network provides a place customers can use to test their integrations and a place where node providers can setup their node and familiarize themselves with the requirements and procedures involved with running a storage node.

## Difference between the test network and the staging network

The _staging_ network (warm-staging) is updated on every commit to main. It ensures builds pass and infrastructure remains in a deployable state. It does not however, guarantee all systems in the network integrate well together. It may also contain partial functionality or breaking API changes that clients do not yet account for.

We SHOULD update the test network at the same time as the production network, however updates to the test network are RECOMMENDED to precede production network updates by at least 24 hours.

## Storage nodes

Storacha will run 3 storage nodes on the network to ensure there are always enough nodes available for upload and replication. These nodes will have limited capacity in order to keep running costs as low as possible.

## Periodic reset

We will setup automation to clear the network every month on the 1st. This helps keep costs low and indicates to external users of the network that it is not suitable for production use.

## Open questions

* How do we signal a push to the test network? An RC tag?
