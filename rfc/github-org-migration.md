# Github Org Migration

![status:draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [João Andrade](https://github.com/joaosa), [Protocol Labs](https://protocol.ai/)

## Authors

- [João Andrade](https://github.com/joaosa), [Protocol Labs](https://protocol.ai/)

## Abstract

Formalizes a process of migrating the [Filecoin Saturn](https://github.com/filecoin-saturn), [NFT.Storage](https://github.com/nftstorage), and [Web3 Storage](https://github.com/web3-storage) orgs into [w3s](https://github.com/w3s-project).

Goals:
- To ensure a common understanding on how repositories should be migrated on a case-by-case basis;
- To achieve cost-savings across orgs by only billing a single one;
- To ensure all repo-dependant systems continue operating transparently;
    - i.e. to provide recommendations/fixes to make the migration process as smooth and bulletproof as possible;
    - If such a goal is not possible, then the secondary goal is to provide the necessary changes to either prevent the system from stopping its operation or to recover it.
- To consolidate repos while minimizing maintenance complexity;
    - e.g. keeping project tracking as simple as possible.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

Source orgs: [Filecoin Saturn](https://github.com/filecoin-saturn), [NFT.Storage](https://github.com/nftstorage), [Web3 Storage](https://github.com/web3-storage).
Final org: [w3s](https://github.com/w3s-project).

Migration is defined as either:
- transferring repo ownership from a source org into the final org;
- forking a source repo in the source org into the final org.

Local Migration is defined as the act of moving a repo into the post-migration filesystem path while ensuring its [Git remote](https://git-scm.com/docs/git-remote) matches a path in the final org.

## Out of scope

- Tidying up local Git state post-migration (i.e. removing old remotes, and branches);
- Defining a standard method to authenticate with Github;
- Settling on a specific Github billing plan;

## Migration consequences and Failure scenarios

According to [Transferring a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/transferring-a-repository): 

> "When you transfer a repository, its issues, pull requests, wiki, stars, and watchers are also transferred. If the transferred repository contains webhooks, services, secrets, or deploy keys, they will remain associated after the transfer is complete. Git information about commits, including contributions, is preserved.".
> (...)
> "All links to the previous repository location are automatically redirected to the new location. When you use git clone, git fetch, or git push on a transferred repository, these commands will redirect to the new repository location or URL. However, to avoid confusion, we strongly recommend updating any existing local clones to point to the new repository URL."

This ensures migration disruptions are minimal and that we can keep operating normally and migrate on a repo-by-repo basis. Still, it is crucial to look at each repo individually to ascertain migration consequences which may impact other systems outside of Github and our local development machines.
Additionally, it is RECOMMENDED to perform the [post-migration instructions](#post-migration-instructions) below.

CI pipelines SHALL be individually checked to make sure nothing breaks or relies on a particular repo naming structure.

## Repository Migration Overview

**Note: This section is under construction and it is where the bulk of work of this spec will go**

We have a total of 203 candidate repositories to migrate.
We SHOULD migrate repos one-by-one with an opportunistic approach after we have established it is safe to do so.
Once a repository is successfully migrated, it will be marked as [x].

The following repos lists were computed with: `curl -Ls -H "Accept: application/vnd.github+json" -H "Authorization: Bearer $GITHUB_TOKEN" -H "X-GitHub-Api-Version: 2022-11-28" https://api.github.com/orgs/$ORG/repos\?per_page\=100\&page\=$PAGE | jq -r '.[] | "- [ ] [" + .name + "](" + .html_url + ")"' |`, where `$ORG ∈ {"web3-storage", "nftstorage", "filecoin-saturn"}`, `GITHUB_TOKEN` is a token with access to all orgs, and `$PAGE ∈ [1,2]`. Further reference [here](https://docs.github.com/en/rest/repos/repos?apiVersion=2022-11-28#list-organization-repositories).

### [Web3 Storage](https://github.com/web3-storage)

- [ ] [ipfs-car](https://github.com/web3-storage/ipfs-car)
- [ ] [web3.storage](https://github.com/web3-storage/web3.storage)
- [ ] [web3-file](https://github.com/web3-storage/web3-file)
- [ ] [go-w3s-client](https://github.com/web3-storage/go-w3s-client)
- [ ] [wrangler-action](https://github.com/web3-storage/wrangler-action)
- [ ] [add-to-web3](https://github.com/web3-storage/add-to-web3)
- [ ] [files-from-path](https://github.com/web3-storage/files-from-path)
- [ ] [multipart-parser](https://github.com/web3-storage/multipart-parser)
- [ ] [example-image-gallery](https://github.com/web3-storage/example-image-gallery)
- [ ] [broker](https://github.com/web3-storage/broker)
- [ ] [web3-schema](https://github.com/web3-storage/web3-schema)
- [ ] [example-forum-dapp](https://github.com/web3-storage/example-forum-dapp)
- [ ] [fauna-dump](https://github.com/web3-storage/fauna-dump)
- [ ] [db-migration-pipeline](https://github.com/web3-storage/db-migration-pipeline)
- [ ] [fauna-dumpify](https://github.com/web3-storage/fauna-dumpify)
    - Old DB migrations for old products;
    - It SHOULD be archived;
    - It SHOULD NOT be migrated.
- [ ] [admin.storage](https://github.com/web3-storage/admin.storage)
- [ ] [hydra-booster](https://github.com/web3-storage/hydra-booster)
- [ ] [ipns-publisher](https://github.com/web3-storage/ipns-publisher)
- [ ] [specs](https://github.com/web3-storage/specs)
- [ ] [consul-cluster-go-ipfs](https://github.com/web3-storage/consul-cluster-go-ipfs)
- [ ] [repin-unpinned-cids](https://github.com/web3-storage/repin-unpinned-cids)
- [ ] [parse-link-header](https://github.com/web3-storage/parse-link-header)
- [ ] [js-replica](https://github.com/web3-storage/js-replica)
- [ ] [w3up-archived](https://github.com/web3-storage/w3up-archived)
- [ ] [upload-protocol](https://github.com/web3-storage/upload-protocol)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [ucanto](https://github.com/web3-storage/ucanto)
- [ ] [w3up](https://github.com/web3-storage/w3up)
- [ ] [ucanto-name-system](https://github.com/web3-storage/ucanto-name-system)
- [ ] [alt-gateway](https://github.com/web3-storage/alt-gateway)
- [ ] [link.web3.storage](https://github.com/web3-storage/link.web3.storage)
- [ ] [dag-check](https://github.com/web3-storage/dag-check)
- [ ] [pickup](https://github.com/web3-storage/pickup)
- [ ] [workers](https://github.com/web3-storage/workers)
- [ ] [minibus](https://github.com/web3-storage/minibus)
- [ ] [w3name](https://github.com/web3-storage/w3name)
- [ ] [reads](https://github.com/web3-storage/reads)
- [ ] [claudio](https://github.com/web3-storage/claudio)
- [ ] [minibus-client](https://github.com/web3-storage/minibus-client)
- [ ] [w3link](https://github.com/web3-storage/w3link)
- [ ] [dagula](https://github.com/web3-storage/dagula)
- [ ] [dagula-gateway](https://github.com/web3-storage/dagula-gateway)
- [ ] [fast-unixfs-exporter](https://github.com/web3-storage/fast-unixfs-exporter)
- [ ] [telemetry-grafana-agent](https://github.com/web3-storage/telemetry-grafana-agent)
- [ ] [handlebars.js](https://github.com/web3-storage/handlebars.js)
- [ ] [w3up-client](https://github.com/web3-storage/w3up-client)
- [ ] [w3up-client-examples](https://github.com/web3-storage/w3up-client-examples)
- [ ] [cf-management](https://github.com/web3-storage/cf-management)
- [ ] [js-libp2p-multistream-select](https://github.com/web3-storage/js-libp2p-multistream-select)
- [ ] [w3up-client-components](https://github.com/web3-storage/w3up-client-components)
- [ ] [w3up-cli](https://github.com/web3-storage/w3up-cli)
- [ ] [goodbits](https://github.com/web3-storage/goodbits)
- [ ] [linkdex](https://github.com/web3-storage/linkdex)
- [ ] [linkdex-api](https://github.com/web3-storage/linkdex-api)
- [ ] [w3ui](https://github.com/web3-storage/w3ui)
- [ ] [web3.storage-content](https://github.com/web3-storage/web3.storage-content)
- [ ] [w3ui-website](https://github.com/web3-storage/w3ui-website)
- [ ] [attach-write-to-read](https://github.com/web3-storage/attach-write-to-read)
- [ ] [sigv4](https://github.com/web3-storage/sigv4)
- [ ] [ipfs-path](https://github.com/web3-storage/ipfs-path)
- [ ] [toucan-js](https://github.com/web3-storage/toucan-js)
- [ ] [freeway](https://github.com/web3-storage/freeway)
- [ ] [w3console](https://github.com/web3-storage/w3console)
- [ ] [gateway-lib](https://github.com/web3-storage/gateway-lib)
- [ ] [diffusionbee-stable-diffusion-ui](https://github.com/web3-storage/diffusionbee-stable-diffusion-ui)
- [ ] [ai-artwork-uploader](https://github.com/web3-storage/ai-artwork-uploader)
- [ ] [sst-monorepo](https://github.com/web3-storage/sst-monorepo)
- [ ] [w3infra](https://github.com/web3-storage/w3infra)
    - Will need to check what happens to the [seed.run](https://seed.run/) infra with a new mini project (SST + seed.run) in the new org and migrate this repo there.
- [ ] [w3notes](https://github.com/web3-storage/w3notes)
- [ ] [w3ui-swc-minify-bug](https://github.com/web3-storage/w3ui-swc-minify-bug)
- [ ] [file-space](https://github.com/web3-storage/file-space)
- [ ] [pickup-e2e-tests](https://github.com/web3-storage/pickup-e2e-tests)
- [ ] [w3cli](https://github.com/web3-storage/w3cli)
- [ ] [aws-tag-audit](https://github.com/web3-storage/aws-tag-audit)
- [ ] [private](https://github.com/web3-storage/private)
- [ ] [pail](https://github.com/web3-storage/pail)
- [ ] [w3link-csp-report-api](https://github.com/web3-storage/w3link-csp-report-api)
- [ ] [backup](https://github.com/web3-storage/backup)
- [ ] [w3filecoin-infra](https://github.com/web3-storage/w3filecoin-infra)
    - Will need to check what happens to the [seed.run](https://seed.run/) infra with a new mini project (SST + seed.run) in the new org and migrate this repo there.
- [ ] [w3up-docs](https://github.com/web3-storage/w3up-docs)
- [ ] [backup-infra](https://github.com/web3-storage/backup-infra)
- [ ] [w3clock](https://github.com/web3-storage/w3clock)
- [ ] [dagcargo-r2-presigned-download-url-gen](https://github.com/web3-storage/dagcargo-r2-presigned-download-url-gen)
- [ ] [blake3-multihash](https://github.com/web3-storage/blake3-multihash)
- [ ] [car-block-validator](https://github.com/web3-storage/car-block-validator)
- [ ] [backup-progress](https://github.com/web3-storage/backup-progress)
- [ ] [secrets](https://github.com/web3-storage/secrets)
- [ ] [carpark-backfill](https://github.com/web3-storage/carpark-backfill)
- [ ] [ipni](https://github.com/web3-storage/ipni)
- [ ] [dealer](https://github.com/web3-storage/dealer)
- [ ] [carpark-bucket-diff](https://github.com/web3-storage/carpark-bucket-diff)
- [ ] [data-segment](https://github.com/web3-storage/data-segment)
- [ ] [gendex](https://github.com/web3-storage/gendex)
- [ ] [autobahn](https://github.com/web3-storage/autobahn)
- [ ] [dag.w3s.link](https://github.com/web3-storage/dag.w3s.link)
- [ ] [w3admin](https://github.com/web3-storage/w3admin)
- [ ] [gendex-consumer](https://github.com/web3-storage/gendex-consumer)
- [ ] [content-claims](https://github.com/web3-storage/content-claims)
- [ ] [sync-multihash-sha2](https://github.com/web3-storage/sync-multihash-sha2)
- [ ] [hoverboard](https://github.com/web3-storage/hoverboard)
- [ ] [access-proxy](https://github.com/web3-storage/access-proxy)
- [ ] [RFC](https://github.com/web3-storage/RFC)
- [ ] [loki-tail-worker](https://github.com/web3-storage/loki-tail-worker)
- [ ] [content-claims-gql](https://github.com/web3-storage/content-claims-gql)
- [ ] [sha256it](https://github.com/web3-storage/sha256it)
- [ ] [migrate-block-index](https://github.com/web3-storage/migrate-block-index)
- [ ] [fr32-sha2-256-trunc254-padded-binary-tree-multihash](https://github.com/web3-storage/fr32-sha2-256-trunc254-padded-binary-tree-multihash)
- [ ] [migrate-block-index-infra](https://github.com/web3-storage/migrate-block-index-infra)
- [ ] [w3stat](https://github.com/web3-storage/w3stat)
- [ ] [go-ucanto](https://github.com/web3-storage/go-ucanto)
- [ ] [pieces](https://github.com/web3-storage/pieces)
- [ ] [console](https://github.com/web3-storage/console)
- [ ] [one-webcrypto](https://github.com/web3-storage/one-webcrypto)
- [ ] [www](https://github.com/web3-storage/www)
- [ ] [learnyouw3up](https://github.com/web3-storage/learnyouw3up)
- [ ] [docs](https://github.com/web3-storage/docs)
- [ ] [go-w3up](https://github.com/web3-storage/go-w3up)
- [ ] [migrate-to-w3up](https://github.com/web3-storage/migrate-to-w3up)
- [ ] [piece-compute-worker](https://github.com/web3-storage/piece-compute-worker)
- [ ] [RFC.staging](https://github.com/web3-storage/RFC.staging)

### [NFT.Storage](https://github.com/nftstorage)

- [ ] [nft.storage](https://github.com/nftstorage/nft.storage)
- [ ] [go-client](https://github.com/nftstorage/go-client)
- [ ] [java-client](https://github.com/nftstorage/java-client)
- [ ] [php-client](https://github.com/nftstorage/php-client)
- [ ] [python-client](https://github.com/nftstorage/python-client)
- [ ] [ruby-client](https://github.com/nftstorage/ruby-client)
- [ ] [rust-client](https://github.com/nftstorage/rust-client)
- [ ] [eip721-subgraph](https://github.com/nftstorage/eip721-subgraph)
- [ ] [ipfs-cluster](https://github.com/nftstorage/ipfs-cluster)
- [ ] [nft.storage-tools](https://github.com/nftstorage/nft.storage-tools)
- [ ] [js-client](https://github.com/nftstorage/js-client)
- [ ] [dagcargo](https://github.com/nftstorage/dagcargo)
- [ ] [carbites](https://github.com/nftstorage/carbites)
- [ ] [carbites-cli](https://github.com/nftstorage/carbites-cli)
- [ ] [nft.storage-example-next.js](https://github.com/nftstorage/nft.storage-example-next.js)
- [ ] [ipnftx](https://github.com/nftstorage/ipnftx)
- [ ] [nfts-growth-initiatives](https://github.com/nftstorage/nfts-growth-initiatives)
- [ ] [consul-cluster-go-ipfs](https://github.com/nftstorage/consul-cluster-go-ipfs)
- [ ] [metaplex-auth](https://github.com/nftstorage/metaplex-auth)
- [ ] [ipnft](https://github.com/nftstorage/ipnft)
- [ ] [backup](https://github.com/nftstorage/backup)
- [ ] [gateway-load-simulator](https://github.com/nftstorage/gateway-load-simulator)
- [ ] [gateway-read-test-lambda](https://github.com/nftstorage/gateway-read-test-lambda)
- [ ] [ucan.storage](https://github.com/nftstorage/ucan.storage)
- [ ] [etl-dotstorage](https://github.com/nftstorage/etl-dotstorage)
- [ ] [checkup](https://github.com/nftstorage/checkup)
- [ ] [niftysave](https://github.com/nftstorage/niftysave)
- [ ] [nft-upload-tools](https://github.com/nftstorage/nft-upload-tools)
- [ ] [assemble-cars-lambda](https://github.com/nftstorage/assemble-cars-lambda)
- [ ] [ipfs-check](https://github.com/nftstorage/ipfs-check)
- [ ] [warm-cache-lambda](https://github.com/nftstorage/warm-cache-lambda)
- [ ] [nftup](https://github.com/nftstorage/nftup)
- [ ] [cloudflare-analytics-prometheus-exporter](https://github.com/nftstorage/cloudflare-analytics-prometheus-exporter)
- [ ] [nftstorage.link](https://github.com/nftstorage/nftstorage.link)
- [ ] [react-nftstorage.link-fallback-example](https://github.com/nftstorage/react-nftstorage.link-fallback-example)
- [ ] [nftstorage-service-worker](https://github.com/nftstorage/nftstorage-service-worker)
- [ ] [test-seed.run](https://github.com/nftstorage/test-seed.run)
- [ ] [dotstorage-db-admin](https://github.com/nftstorage/dotstorage-db-admin)
- [ ] [dagcargo-bucket](https://github.com/nftstorage/dagcargo-bucket)
- [ ] [nft.storage-content](https://github.com/nftstorage/nft.storage-content)
- [ ] [nftdotstorage](https://github.com/nftstorage/nftdotstorage)

### [Filecoin Saturn](https://github.com/filecoin-saturn)

Migration here refers to transferring a repo into the final org.

Recommendation: all repos which end up getting migrated should have their name changed and prefixed with `saturn-` if this prefix isn't part of their names already.

- [ ] [DEPRECATED-saturn](https://github.com/filecoin-saturn/DEPRECATED-saturn)
    - An historical artefact of an initial Filecoin Saturn implementation. 
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [browser-client](https://github.com/filecoin-saturn/browser-client)
    - The Saturn Browser Client is a service worker that serves websites' CID requests with CAR files. CAR files are verifiable, which is a requirement when retrieving content in a trustless manner from community hosted Saturn Nodes.
    - It MUST be migrated.
- [ ] [L1-node](https://github.com/filecoin-saturn/L1-node)
    - Saturn L1 nodes are CDN edge caches in the outermost layer of the Filecoin Saturn network. L1 nodes serve CAR files to retrieval clients as requested by their CIDs. Cache misses are served by the IPFS Network and Filecoin Storage Providers.
    - Any migration will require a network-wide update and changes to the self-update scripts which directly depend on the repo name. This SHOULD be done immediately after the migration.
    - It MUST be migrated.
- [ ] [homepage](https://github.com/filecoin-saturn/homepage)
    - Saturn's homepage
    - It MUST be migrated.
- [ ] [terraform](https://github.com/filecoin-saturn/terraform)
    - Saturn's terraform IaC.
    - It MUST be migrated.
- [ ] [lambdas](https://github.com/filecoin-saturn/lambdas)
    - Had misc lambdas for Saturn's operation. These store bandwidth logs, provide a metrics API, calculate FIL earnings, run fraud analysis on logs, and serve JWTs.
    - It MUST be migrated.
- [ ] [DEPRECATED-media](https://github.com/filecoin-saturn/DEPRECATED-media)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [orchestrator](https://github.com/filecoin-saturn/orchestrator)
    - Saturn's centralised network orchestrator.
    - It MUST be migrated.
- [ ] [L2-node](https://github.com/filecoin-saturn/L2-node)
    - Legacy repo for Saturn's l2 nodes which are no longer relevant.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [L1-dashboard](https://github.com/filecoin-saturn/L1-dashboard)
    - A dashboard UI for Filecoin Saturn's L1 node. Hosted at https://dashboard.saturn.tech.
    - It MUST be migrated.
- [ ] [metrics-dashboard](https://github.com/filecoin-saturn/metrics-dashboard)
    - Hosts the metric aggregation code for Saturn's Grafana.
    - It MUST be migrated.
- [ ] [http-testing](https://github.com/filecoin-saturn/http-testing)
    - HTTP3 testing for nginx and Saturn.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [kubo](https://github.com/filecoin-saturn/kubo)
    - A Kubo fork which AFAICT isn't needed.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [js-client](https://github.com/filecoin-saturn/js-client)
    - The official JavaScript client for Filecoin Saturn. Used by [browser-client](https://github.com/filecoin-saturn/browser-client).
    - It MUST be migrated.
- [ ] [roadmap](https://github.com/filecoin-saturn/roadmap)
    - Saturn's old project roadmap.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [contracts](https://github.com/filecoin-saturn/contracts)
    - Saturn's smart contract tooling and the cli to deploy payouts.
    - It MUST be migrated.
- [ ] [zk-fraud](https://github.com/filecoin-saturn/zk-fraud)
    - A tentative project to create a simple zk circuit which replicates the fraud detection algorithm on node logs
    - AFAICT this is not being used
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [payouts-website](https://github.com/filecoin-saturn/payouts-website)
    - The repo behind https://payouts.saturn.tech, the website where node operators interact with Saturn's payout smart contract to receive their FIL payouts for network contributions.
    - It MUST be migrated.
- [ ] [caboose](https://github.com/filecoin-saturn/caboose)
    - A remote-car blockstore which provides a blockstore interface over a dynamic set of remote car providers. This was heavily used during Project Rhea.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [ansible](https://github.com/filecoin-saturn/ansible))
    - The tools to provision, manage and configure Saturn trusted/core nodes.
    - It MUST be migrated.
- [ ] [explorer](https://github.com/filecoin-saturn/explorer)
    - A geospatial visualization for Saturn network stats. Live at https://explorer.saturn.tech.
    - It MUST be migrated.
- [ ] [saturn-analysis](https://github.com/filecoin-saturn/saturn-analysis)
    - Documents the fraud and payment analysis performed for the Saturn Network.
    - It MUST be migrated.
- [ ] [nginx-car-range](https://github.com/filecoin-saturn/nginx-car-range)
    - Nginx plugin for filtering range requests from CAR files.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [saturn-docs](https://github.com/filecoin-saturn/saturn-docs)
    - Saturn documentation hosted by Super at docs.saturn.tech.
    - It MUST be migrated.
- [ ] [rs-fevm-utils](https://github.com/filecoin-saturn/rs-fevm-utils)
    - Repo for fEVM related utility functions in rust which is a depdency of [contracts](https://github.com/filecoin-saturn/contracts).
    - It MUST be migrated.
- [ ] [fevm-deployment-tutorial](https://github.com/filecoin-saturn/fevm-deployment-tutorial)
    - Related to and a PoC to [contracts](https://github.com/filecoin-saturn/contracts) deployment on fEVM using [foundry](https://github.com/foundry-rs/foundry).
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [L1-replay](https://github.com/filecoin-saturn/L1-replay)
    - Same as [L1-replay-go](https://github.com/filecoin-saturn/L1-replay-go).
    - It SHOULD be migrated if [L1-replay-go](https://github.com/filecoin-saturn/L1-replay-go) is not migrated.
    - It SHOULD be archived if [L1-replay-go](https://github.com/filecoin-saturn/L1-replay-go) is migrated.
- [ ] [L1-replay-go](https://github.com/filecoin-saturn/L1-replay-go)
    - Same as [L1-replay](https://github.com/filecoin-saturn/L1-replay).
    - It SHOULD be migrated if [L1-replay](https://github.com/filecoin-saturn/L1-replay) is not migrated.
    - It SHOULD be archived if [L1-replay](https://github.com/filecoin-saturn/L1-replay) is migrated.
- [ ] [size-diff-tool](https://github.com/filecoin-saturn/size-diff-tool)
    - AFAICT it's a tool to measure the size of [orchestrator](https://github.com/filecoin-saturn/orchestrator). It has not documentation or README, so I SHOULD be dropped.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [misc](https://github.com/filecoin-saturn/misc)
    - A collection of miscellaneous Saturn tools, scripts, and snippets that don't warrant their own repos.
    - It MUST be migrated.
- [ ] [saturn-demo](https://github.com/filecoin-saturn/saturn-demo)
    - A demo Github Pages website which uses the Saturn network and it's helpful to demo it.
    - It MUST be migrated.
- [ ] [onion](https://github.com/filecoin-saturn/onion)
    - Part of troubleshooting Project Rhea.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [saturn-revenue](https://github.com/filecoin-saturn/saturn-revenue)
    - A motivational repo.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [eslint-config](https://github.com/filecoin-saturn/eslint-config)
    - Saturn's Javascript eslint config. I think it can be archived and used as a ref as whatever standard we use needs to belong to the team.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [alex-prod](https://github.com/filecoin-saturn/alex-prod)
    - A test repo for @alexprod.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [cassiopeia](https://github.com/filecoin-saturn/cassiopeia)
    - An example approach to start [pluto](https://github.com/filecoin-saturn/pluto).
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [js-ipfs-unixfs](https://github.com/filecoin-saturn/js-ipfs-unixfs)
    - AFAICT it's a custom version which is bundled with [browser-client](https://github.com/filecoin-saturn/browser-client).
    - It MUST be migrated.
- [ ] [pluto](https://github.com/filecoin-saturn/pluto)
    - The unfinished golang migration of [L1-node](https://github.com/filecoin-saturn/L1-node). We can always pick it up from archival if we need it.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [prod-issues](https://github.com/filecoin-saturn/prod-issues)
    - Saturn production issues tracker. It's no longer being used.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [portal](https://github.com/filecoin-saturn/portal)
    - Saturn's customer portal.
    - It MUST be migrated.
- [ ] [brand-assets](https://github.com/filecoin-saturn/brand-assets)
    - WIP brand assets for the Saturn brand.
    - It MUST be migrated.
- [ ] [project-tracking](https://github.com/filecoin-saturn/project-tracking)
    - Project tracking for Saturn.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
    - [The project board](https://github.com/orgs/filecoin-saturn/projects/1/views/1) SHOULD be checked for anything important.
- [ ] [compare-payouts](https://github.com/filecoin-saturn/compare-payouts)
    - The companion tool to process [contracts](https://github.com/filecoin-saturn/contracts) payment CSVs with credits.
    - It MUST be migrated.

## Post-migration instructions

### Local Migration

The goal of this section is to provide a concise and simple way to operate on a given repo post-migration without losing any local changes. 
This is provided in the form of shell commands (more specifically using `bash` and [GNU findutils](https://www.gnu.org/software/findutils)).

Exact repo path are provided as examples and may not result in a precise command for everyone. To account for this, `$PATH_PREFIX` should equal the path prefix that is required to reach a given repo from your current directory. For instance, while writing this, this means `PATH_PREFIX=~/ghq/github.com`.

A given `ORG` is assumed to exist in the one's filesystem beforehand. If not a [git clone](https://git-scm.com/docs/git-clone) targeting `https://github.com/w3s-project/$REPO` is RECOMMENDED (see the [Repository Overview](#repository-migration-overview) for specific values for `REPO`).

#### Partial org migration

In this scenario, one is attempting to do a local migration on a single or set of repos.
We assume your Git remote is called `origin`.

1. Check your remote for the repo to make sure you're not missing anything: `git -C $REPO remote -v`;
2. Set the remote to the final org: `git -C "$PATH_REFIX/$ORG/$REPO" remote set-url origin "git@github.com:w3s-project/$REPO"`.

#### Full org migration

**Note: Run with caution, as this section is being validated**

In this scenario, one is attempting to do a local migration on all local repos for a particular `ORG`.
We assume your Git remote is called `origin` and that ssh authentication is the desired method (if you want to switch authentication look [here](https://docs.github.com/en/get-started/getting-started-with-git/managing-remote-repositories#switching-remote-urls-from-https-to-ssh)).

1. Check your remotes for all repos within an `ORG` to make sure you're not missing anything: `find "$PATH_PREFIX/$ORG" -maxdepth 1 -type d ! -path "$PATH_PREFIX/$ORG" | xargs -pI{} git -C {} remote -v`;
2. Move the `ORG` into its new filesystem path: `mv "$PATH_PREFIX/$ORG" "$PATH_PREFIX/w3s-project"`;
3. Set remotes to the final org: `find "$PATH_PREFIX/$ORG" -maxdepth 1 -type d ! -path "$PATH_PREFIX/$ORG" | xargs -pI{} sh -c 'repo="$(basename {})"; git -C {} remote set-url origin "git@github.com:w3s-project/$repo"'`.
