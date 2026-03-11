# Forge demo network

This RFC proposes a stable network for Forge that can be used as a bootstrapping/onboarding playground for customers and an area for live demonstrations given by the Storacha team. The network should be _considered_ a production network as it is mission critical that it remains operational at all times.

The Forge network requires customers to commit to storage capacity up front. They must also integrate the network with their systems/applications. Both of these are significant hurdles to using the production network. The demo network aims to lower that boundary by providing a place where potential customers can try out the network - allowing experimentation to take place and failures to happen without financial consequence. In short, the demo network provides a place customers can use to test their integrations.

The demo network will also be used by the Storacha team in live demonstrations to prospective customers, or in presentations to promote the service.

We MUST ensure the demo network is kept in sync with the production network at all times.

## Difference between the demo network and the staging network

The _staging_ network (warm-staging) is updated on every commit to main. It ensures builds pass and infrastructure remains in a deployable state. It does not however, guarantee all systems in the network integrate well together. It may also contain partial functionality or breaking API changes that clients do not yet account for.

## Storage nodes

Storacha will run 3 storage nodes on the network to ensure there are always enough nodes available for upload and replication. These nodes will have limited capacity in order to keep running costs as low as possible.

## Periodic reset

We will setup automation to clear the network every month on the 1st. This helps keep costs low and indicates to external users of the network that it is not suitable for production use.

Attestations issued by the service MUST expire at the start of the following month.

## Filecoin chain

As with the staging network, the demo network will target the Filecoin Calibnet network.
