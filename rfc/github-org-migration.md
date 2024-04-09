# Github Org Migration

![status:final](https://img.shields.io/badge/status-final-green.svg?style=flat-square)

## Editors

- [João Andrade](https://github.com/joaosa), [Protocol Labs](https://protocol.ai/)

## Authors

- [João Andrade](https://github.com/joaosa), [Protocol Labs](https://protocol.ai/)

## Abstract

Formalizes a process of migrating the [Filecoin Saturn](https://github.com/filecoin-saturn), [NFT.Storage](https://github.com/nftstorage), and [Web3 Storage](https://github.com/web3-storage) orgs into [w3s](https://github.com/w3s-project).

Goals:
- To achieve cost-savings across orgs by only billing a single one;
- To minimize maintenance complexity by consolidating the list of org repos we want to take on;
- To ensure a common understanding on how repositories should be migrated on a case-by-case basis;
- To ensure all repo-dependant systems continue operating transparently;
    - i.e. to provide recommendations/fixes to make the migration process as smooth and bulletproof as possible;
    - If such a goal is not possible, then the secondary goal is to provide the necessary changes to either prevent the system from stopping its operation or to recover it.
- To keep project tracking as simple as possible.
    - i.e. across a single org.

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

CI pipelines were checked to make sure nothing relies on a particular repo naming structure. Github was used and the results were manually checked using the following search commands:

  - `org:filecoin-saturn AND ("git@github.com" OR "https://github.com" OR "https://raw.githubusercontent.com") AND (path:*.yaml OR path:*.yml)`
  - `org:web3-storage  AND ("git@github.com" OR "https://github.com" OR "https://raw.githubusercontent.com") AND (path:*.yaml OR path:*.yml)`

## Repository Migration Overview

New repositories MUST be created in [w3s](https://github.com/w3s-project).

The goal of this section is to migrate repos we are going to be actively working on, so we're able to downgrade source orgs to the Github free tier.
Ideally, migrations should happen when we are working on repos whose purpose overlaps between w3s and the previous project.

We have a total of 193 candidate repositories to migrate (out of a total of 203 of which 10 were already archived).
We SHOULD migrate repos one-by-one with an opportunistic approach after we have established it is safe to do so.
Once the actions for a repository are successfully identified, it will be marked as [x].
If a given repository has no checkbox, than means it was sorted prior to this document.

The following repos lists were computed with: `curl -Ls -H "Accept: application/vnd.github+json" -H "Authorization: Bearer $GITHUB_TOKEN" -H "X-GitHub-Api-Version: 2022-11-28" https://api.github.com/orgs/$ORG/repos\?per_page\=100\&page\=$PAGE | jq -r '.[] | select(.archived|not) | "- [ ] [" + .name + "](" + .html_url + ")"'`, where `$ORG ∈ {"web3-storage", "nftstorage", "filecoin-saturn"}`, `GITHUB_TOKEN` is a token with access to all orgs, and `$PAGE ∈ [1,2]`. Further reference [here](https://docs.github.com/en/rest/repos/repos?apiVersion=2022-11-28#list-organization-repositories).

### [Web3 Storage](https://github.com/web3-storage)

[ ] The [Web3 Storage](https://github.com/web3-storage) SHOULD be downgraded to the Github Team plan for immediate cost savings.

- [ ] [ipfs-car](https://github.com/web3-storage/ipfs-car)
    - CAR client for UnixFS dags. It is heavilty used.
    - It SHOULD be migrated.
- [ ] [web3.storage](https://github.com/web3-storage/web3.storage)
    - Monorepo for the old website. It deploys to Cloudflare Pages.
    - It SHOULD NOT be migrated.
- [web3-file](https://github.com/web3-storage/web3-file)
    - already archived.
- [ ] [go-w3s-client](https://github.com/web3-storage/go-w3s-client)
    - client for the old API.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [wrangler-action](https://github.com/web3-storage/wrangler-action)
    - Zero-config Cloudflare Workers deployment using Wrangler and GitHub Actions. It's being used on some repos liked `reads` and `freeway`. More [here](https://github.com/search?q=org%3Aweb3-storage%20wrangler-action&type=code).
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [add-to-web3](https://github.com/web3-storage/add-to-web3)
    - Github action to put websites on web3.storage.It has probably been updated.
    - It SHOULD be migrated.
- [ ] [files-from-path](https://github.com/web3-storage/files-from-path)
    - A lib for globbing a bunch of files in a dir and returning that as an array of file objects.
    - It SHOULD be migrated.
- [ ] [multipart-parser](https://github.com/web3-storage/multipart-parser)
    - A simple multipart/form-data parser to use with ReadableStreams. It's being used by a fair number of repos. See [here](https://github.com/search?q=org%3Aweb3-storage+multipart-parser&type=code).
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [example-image-gallery](https://github.com/web3-storage/example-image-gallery)
    - A simple example of using Web3.Storage to share images using decentralized storage.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [broker](https://github.com/web3-storage/broker)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [web3-schema](https://github.com/web3-storage/web3-schema)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [example-forum-dapp](https://github.com/web3-storage/example-forum-dapp)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [fauna-dump](https://github.com/web3-storage/fauna-dump)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [db-migration-pipeline](https://github.com/web3-storage/db-migration-pipeline)
    - Related to fauna-dump.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [fauna-dumpify](https://github.com/web3-storage/fauna-dumpify)
    - Old DB migrations for old products;
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [admin.storage](https://github.com/web3-storage/admin.storage)
    - Admin web interface for the old api.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [hydra-booster](https://github.com/web3-storage/hydra-booster)
    - This is a fork.
    - It SHOULD be deleted.
- [ ] [ipns-publisher](https://github.com/web3-storage/ipns-publisher)
    - Create your IPNS records in Javascript and publish them on the IPFS network.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [specs](https://github.com/web3-storage/specs)
    - It SHOULD NOT be archived.
    - It SHOULD be migrated.
- [ ] [consul-cluster-go-ipfs](https://github.com/web3-storage/consul-cluster-go-ipfs)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [repin-unpinned-cids](https://github.com/web3-storage/repin-unpinned-cids)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [parse-link-header](https://github.com/web3-storage/parse-link-header)
    - Old but it should be left be.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [js-replica](https://github.com/web3-storage/js-replica)
    - Empty repo.
    - It SHOULD be deleted.
- [ ] [w3up-archived](https://github.com/web3-storage/w3up-archived)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [upload-protocol](https://github.com/web3-storage/upload-protocol)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [ucanto](https://github.com/web3-storage/ucanto)
    - UCAN encoding implication for creating UCANs and sending them to a service and verifying delegations.
    - It SHOULD NOT be archived.
    - It SHOULD be migrated.
- [ ] [w3up](https://github.com/web3-storage/w3up)
    - Implementation of the protocols and specs repos. The current web3.storage system.
    - It SHOULD NOT be archived.
    - It SHOULD be migrated.
- [ ] [ucanto-name-system](https://github.com/web3-storage/ucanto-name-system)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [alt-gateway](https://github.com/web3-storage/alt-gateway)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [link.web3.storage](https://github.com/web3-storage/link.web3.storage)
    - Tests done in the team week in Miami to test the service worker deployment.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [dag-check](https://github.com/web3-storage/dag-check)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [pickup](https://github.com/web3-storage/pickup)
    - Used by nft.storage for the pinning service API (only supported by them)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [workers](https://github.com/web3-storage/workers)
    - In use for the workers for things like loki/logs. Check https://github.com/web3-storage/reads/blob/main/packages/edge-gateway/package.json#L21
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [minibus](https://github.com/web3-storage/minibus)
    - There's a deployed service with this but we're not using it in production. A small data service and it works with blocks for CIDs. Works for smaller things and not for big DAGs
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3name](https://github.com/web3-storage/w3name)
    - Used in production for IPNS.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [reads](https://github.com/web3-storage/reads)
    - Frontend for the racing GW system. Also includes a denylist that gets checked before racing. A superset of badbits.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [claudio](https://github.com/web3-storage/claudio)
    - Original MVP for hoverboard.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [minibus-client](https://github.com/web3-storage/minibus-client)
    - The cli for minibus.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3link](https://github.com/web3-storage/w3link)
    - The production deployed worker for w3s.link.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [dagula](https://github.com/web3-storage/dagula)
    - Used by freeway and hoverboard. lib and cli for fetching a DAG from a remote peer.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [dagula-gateway](https://github.com/web3-storage/dagula-gateway)
    - Experiment to back a GW with Elastic IPFS over Bitswap, but it didn't work out with the amount of traffic.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [fast-unixfs-exporter](https://github.com/web3-storage/fast-unixfs-exporter)
    - already archived.
- [ ] [telemetry-grafana-agent](https://github.com/web3-storage/telemetry-grafana-agent)
    - This holds a bunch of our Grafana metrics, including w3up, gateways, cloudflare metrics, etc.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [handlebars.js](https://github.com/web3-storage/handlebars.js)
    - Fork used in the gateway and used for listing files.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [w3up-client](https://github.com/web3-storage/w3up-client)
    - already archived.
- [ ] [w3up-client-examples](https://github.com/web3-storage/w3up-client-examples)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [cf-management](https://github.com/web3-storage/cf-management)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [js-libp2p-multistream-select](https://github.com/web3-storage/js-libp2p-multistream-select)
    - Reduces roundtrips. do you speak this with data right away so one less roundtrip. IPFS network peers do speak same protocols. [It's no longer used](https://github.com/search?q=org%3Aweb3-storage+js-libp2p-multistream-select&type=code).
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3up-client-components](https://github.com/web3-storage/w3up-client-components)
    - It SHOULD be deleted.
- [w3up-cli](https://github.com/web3-storage/w3up-cli)
    - already archived.
- [ ] [goodbits](https://github.com/web3-storage/goodbits)
    - if your CID got accidentally banned, you can submit an issue and say it's good.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [linkdex](https://github.com/web3-storage/linkdex)
    - Lib for the api.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [linkdex-api](https://github.com/web3-storage/linkdex-api)
    - Indexes and ensures DAGs have the right blocks and if a DAG is complete. Used by pickup.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3ui](https://github.com/web3-storage/w3ui)
    - UI components. Lib for devs to build UI with web3.storage.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [web3.storage-content](https://github.com/web3-storage/web3.storage-content)
    - Blog for the old website.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3ui-website](https://github.com/web3-storage/w3ui-website)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [attach-write-to-read](https://github.com/web3-storage/attach-write-to-read)
    - already archived.
- [ ] [sigv4](https://github.com/web3-storage/sigv4)
    - Used by w3up to generate presigned urls.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [ipfs-path](https://github.com/web3-storage/ipfs-path)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [toucan-js](https://github.com/web3-storage/toucan-js)
    - Fork for talking to Sentry in CF workers and it's not used anymore. Upstream didn't support error-cause property and it was needed.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [freeway](https://github.com/web3-storage/freeway)
    - HTTP GW which reads CAR files and serves data. w3s.link and ntfstorage.link talk to this.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [w3console](https://github.com/web3-storage/w3console)
    - already archived.
- [ ] [gateway-lib](https://github.com/web3-storage/gateway-lib)
    - Used by freeway and by dagula-gw.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [diffusionbee-stable-diffusion-ui](https://github.com/web3-storage/diffusionbee-stable-diffusion-ui)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [ai-artwork-uploader](https://github.com/web3-storage/ai-artwork-uploader)
    - It should be left behind as a demo.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [sst-monorepo](https://github.com/web3-storage/sst-monorepo)
    - It SHOULD be deleted.
- [ ] [w3infra](https://github.com/web3-storage/w3infra)
    - [ ] Will need to check what happens to the [seed.run](https://seed.run/) infra with a new mini project (SST + seed.run) in the new org and migrate this repo there.
    - It SHOULD NOT be archived.
    - It SHOULD be migrated.
- [ ] [w3notes](https://github.com/web3-storage/w3notes)
    - Example app.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3ui-swc-minify-bug](https://github.com/web3-storage/w3ui-swc-minify-bug)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [file-space](https://github.com/web3-storage/file-space)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [pickup-e2e-tests](https://github.com/web3-storage/pickup-e2e-tests)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3cli](https://github.com/web3-storage/w3cli)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [aws-tag-audit](https://github.com/web3-storage/aws-tag-audit)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [private](https://github.com/web3-storage/private)
    - This is not used.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [pail](https://github.com/web3-storage/pail)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3link-csp-report-api](https://github.com/web3-storage/w3link-csp-report-api)
    - Leave behind for CSP.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [backup](https://github.com/web3-storage/backup)
    - From when moving from cluster to s3.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3filecoin-infra](https://github.com/web3-storage/w3filecoin-infra)
    - Will need to check what happens to the [seed.run](https://seed.run/) infra with a new mini project (SST + seed.run) in the new org and migrate this repo there.
    - It SHOULD NOT be archived.
    - It SHOULD be migrated.
- [w3up-docs](https://github.com/web3-storage/w3up-docs)
    - already archived.
- [ ] [backup-infra](https://github.com/web3-storage/backup-infra)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3clock](https://github.com/web3-storage/w3clock)
    - Service used in production.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [dagcargo-r2-presigned-download-url-gen](https://github.com/web3-storage/dagcargo-r2-presigned-download-url-gen)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [blake3-multihash](https://github.com/web3-storage/blake3-multihash)
    - [ ] Ask @gozala if it can be archived.
- [ ] [car-block-validator](https://github.com/web3-storage/car-block-validator)
    - Used in production.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [backup-progress](https://github.com/web3-storage/backup-progress)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [secrets](https://github.com/web3-storage/secrets)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [carpark-backfill](https://github.com/web3-storage/carpark-backfill)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [ipni](https://github.com/web3-storage/ipni)
    - To talk to IPNI.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [dealer](https://github.com/web3-storage/dealer)
    - already archived.
- [ ] [carpark-bucket-diff](https://github.com/web3-storage/carpark-bucket-diff)
    - It SHOULD be deleted.
- [ ] [data-segment](https://github.com/web3-storage/data-segment)
    - This lib is being used.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [gendex](https://github.com/web3-storage/gendex)
    - An experiment.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [autobahn](https://github.com/web3-storage/autobahn)
    - GW in AWS.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [dag.w3s.link](https://github.com/web3-storage/dag.w3s.link)
    - Trustless GW for Saturn.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3admin](https://github.com/web3-storage/w3admin)
    - Admin interface for the new UCAN APIs.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [gendex-consumer](https://github.com/web3-storage/gendex-consumer)
    - It can be archived as it didn't quite work out.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [content-claims](https://github.com/web3-storage/content-claims)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [sync-multihash-sha2](https://github.com/web3-storage/sync-multihash-sha2)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [hoverboard](https://github.com/web3-storage/hoverboard)
    - Bitswap in CF workers used in production.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [access-proxy](https://github.com/web3-storage/access-proxy)
    - [ ] Ask @travis if it can be archived.
- [ ] [RFC](https://github.com/web3-storage/RFC)
    - It SHOULD NOT be archived.
    - It SHOULD be migrated.
- [ ] [loki-tail-worker](https://github.com/web3-storage/loki-tail-worker)
    - Worker to tail logs.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [content-claims-gql](https://github.com/web3-storage/content-claims-gql)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [sha256it](https://github.com/web3-storage/sha256it)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [migrate-block-index](https://github.com/web3-storage/migrate-block-index)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [fr32-sha2-256-trunc254-padded-binary-tree-multihash](https://github.com/web3-storage/fr32-sha2-256-trunc254-padded-binary-tree-multihash)
    - Lib used for piece CIDs.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [migrate-block-index-infra](https://github.com/web3-storage/migrate-block-index-infra)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [w3stat](https://github.com/web3-storage/w3stat)
    - To diagnose CID problems.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [go-ucanto](https://github.com/web3-storage/go-ucanto)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [pieces](https://github.com/web3-storage/pieces)
    - Cli for devs.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [console](https://github.com/web3-storage/console)
    - Web UI for w3up.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [one-webcrypto](https://github.com/web3-storage/one-webcrypto)
    - Fork which w3up depends upon. See [here](https://github.com/web3-storage/w3up/blob/6fa7797ba6f5c0bbd359c807e257882a7d5d6fb8/packages/access-client/package.json#L113).
- [ ] [www](https://github.com/web3-storage/www)
    - Current website.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [learnyouw3up](https://github.com/web3-storage/learnyouw3up)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [docs](https://github.com/web3-storage/docs)
    - already archived.
- [ ] [go-w3up](https://github.com/web3-storage/go-w3up)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [migrate-to-w3up](https://github.com/web3-storage/migrate-to-w3up)
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [piece-compute-worker](https://github.com/web3-storage/piece-compute-worker)
    - We might need this as a worker to compute Filecoin Pieces.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.
- [ ] [RFC.staging](https://github.com/web3-storage/RFC.staging)
    - It should be moved to the main RFC repo
    - [ ] Move to main RFC repo.
    - It SHOULD NOT be archived.
    - It SHOULD NOT be migrated.

### [NFT.Storage](https://github.com/nftstorage)

The recommendation here is to leave everything as is.

- [x] [nft.storage](https://github.com/nftstorage/nft.storage)
    - the main repo
    - It SHOULD NOT be migrated.
- [x] [go-client](https://github.com/nftstorage/go-client)
    - generated from OpenAPI schemas. We don't know how well they work
    - It SHOULD NOT be migrated.
- [x] [java-client](https://github.com/nftstorage/java-client)
    - generated from OpenAPI schemas. We don't know how well they work
    - It SHOULD NOT be migrated.
- [x] [php-client](https://github.com/nftstorage/php-client)
    - generated from OpenAPI schemas. We don't know how well they work
    - It SHOULD NOT be migrated.
- [x] [python-client](https://github.com/nftstorage/python-client)
    - generated from OpenAPI schemas. We don't know how well they work
    - It SHOULD NOT be migrated.
- [x] [ruby-client](https://github.com/nftstorage/ruby-client)
    - generated from OpenAPI schemas. We don't know how well they work
    - It SHOULD NOT be migrated.
- [x] [rust-client](https://github.com/nftstorage/rust-client)
    - generated from OpenAPI schemas. We don't know how well they work
    - It SHOULD NOT be migrated.
- [x] [eip721-subgraph](https://github.com/nftstorage/eip721-subgraph)
    - It SHOULD NOT be migrated.
- [x] [ipfs-cluster](https://github.com/nftstorage/ipfs-cluster)
    - It SHOULD NOT be migrated.
- [x] [nft.storage-tools](https://github.com/nftstorage/nft.storage-tools)
    - It SHOULD NOT be migrated.
- [x] [js-client](https://github.com/nftstorage/js-client)
    - generated from OpenAPI schemas. We don't know how well they work
    - It SHOULD NOT be migrated.
- [x] [dagcargo](https://github.com/nftstorage/dagcargo)
    - It SHOULD NOT be migrated.
- [x] [carbites](https://github.com/nftstorage/carbites)
    - It SHOULD NOT be migrated.
- [x] [carbites-cli](https://github.com/nftstorage/carbites-cli)
    - It SHOULD NOT be migrated.
- [x] [nft.storage-example-next.js](https://github.com/nftstorage/nft.storage-example-next.js)
    - It SHOULD NOT be migrated.
- [x] [ipnftx](https://github.com/nftstorage/ipnftx)
    - It SHOULD NOT be migrated.
- [x] [nfts-growth-initiatives](https://github.com/nftstorage/nfts-growth-initiatives)
    - It SHOULD NOT be migrated.
- [x] [consul-cluster-go-ipfs](https://github.com/nftstorage/consul-cluster-go-ipfs)
    - It SHOULD NOT be migrated.
- [x] [metaplex-auth](https://github.com/nftstorage/metaplex-auth)
    - It SHOULD NOT be migrated.
- [x] [ipnft](https://github.com/nftstorage/ipnft)
    - It SHOULD NOT be migrated.
- [x] [backup](https://github.com/nftstorage/backup)
    - It SHOULD NOT be migrated.
- [x] [gateway-load-simulator](https://github.com/nftstorage/gateway-load-simulator)
    - It SHOULD NOT be migrated.
- [x] [gateway-read-test-lambda](https://github.com/nftstorage/gateway-read-test-lambda)
    - already archived.
- [x] [ucan.storage](https://github.com/nftstorage/ucan.storage)
    - It SHOULD NOT be migrated.
- [x] [etl-dotstorage](https://github.com/nftstorage/etl-dotstorage)
    - It SHOULD NOT be migrated.
- [x] [checkup](https://github.com/nftstorage/checkup)
    - It SHOULD NOT be migrated.
- [x] [niftysave](https://github.com/nftstorage/niftysave)
    - It SHOULD NOT be migrated.
- [x] [nft-upload-tools](https://github.com/nftstorage/nft-upload-tools)
    - It SHOULD NOT be migrated.
- [x] [assemble-cars-lambda](https://github.com/nftstorage/assemble-cars-lambda)
    - It SHOULD NOT be migrated.
- [x] [ipfs-check](https://github.com/nftstorage/ipfs-check)
    - It SHOULD NOT be migrated.
- [x] [warm-cache-lambda](https://github.com/nftstorage/warm-cache-lambda)
    - It SHOULD NOT be migrated.
- [x] [nftup](https://github.com/nftstorage/nftup)
    - It SHOULD NOT be migrated.
- [x] [cloudflare-analytics-prometheus-exporter](https://github.com/nftstorage/cloudflare-analytics-prometheus-exporter)
    - It SHOULD NOT be migrated.
- [x] [nftstorage.link](https://github.com/nftstorage/nftstorage.link)
    - It SHOULD NOT be migrated.
- [x] [react-nftstorage.link-fallback-example](https://github.com/nftstorage/react-nftstorage.link-fallback-example)
    - It SHOULD NOT be migrated.
- [x] [nftstorage-service-worker](https://github.com/nftstorage/nftstorage-service-worker)
    - It SHOULD NOT be migrated.
- [x] [test-seed.run](https://github.com/nftstorage/test-seed.run)
    - It SHOULD NOT be migrated.
- [x] [dotstorage-db-admin](https://github.com/nftstorage/dotstorage-db-admin)
    - It SHOULD NOT be migrated.
- [x] [dagcargo-bucket](https://github.com/nftstorage/dagcargo-bucket)
    - It SHOULD NOT be migrated.
- [x] [nft.storage-content](https://github.com/nftstorage/nft.storage-content)
    - It SHOULD NOT be migrated.
- [x] [nftdotstorage](https://github.com/nftstorage/nftdotstorage)
    - It SHOULD NOT be migrated.

### [Filecoin Saturn](https://github.com/filecoin-saturn)

Migration here refers to transferring a repo into the final org.

Recommendation: all repos which end up getting migrated should have their name changed and prefixed with `saturn-` if this prefix isn't part of their names already.

- [ ] [DEPRECATED-saturn](https://github.com/filecoin-saturn/DEPRECATED-saturn)
    - An historical artefact of an initial Filecoin Saturn implementation. 
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [browser-client](https://github.com/filecoin-saturn/browser-client)
    - The Saturn Browser Client is a service worker that serves websites' CID requests with CAR files. CAR files are verifiable, which is a requirement when retrieving content in a trustless manner from community hosted Saturn Nodes.
    - It SHOULD be migrated.
- [ ] [L1-node](https://github.com/filecoin-saturn/L1-node)
    - Saturn L1 nodes are CDN edge caches in the outermost layer of the Filecoin Saturn network. L1 nodes serve CAR files to retrieval clients as requested by their CIDs. Cache misses are served by the IPFS Network and Filecoin Storage Providers.
    - Any migration will require a network-wide update and changes to the self-update scripts which directly depend on the repo name. This SHOULD be done immediately after the migration.
    - It SHOULD be migrated.
- [ ] [homepage](https://github.com/filecoin-saturn/homepage)
    - Saturn's homepage
    - It SHOULD be migrated.
- [ ] [terraform](https://github.com/filecoin-saturn/terraform)
    - Saturn's terraform IaC.
    - It SHOULD be migrated.
- [ ] [lambdas](https://github.com/filecoin-saturn/lambdas)
    - Had misc lambdas for Saturn's operation. These store bandwidth logs, provide a metrics API, calculate FIL earnings, run fraud analysis on logs, and serve JWTs.
    - It SHOULD be migrated.
- [ ] [DEPRECATED-media](https://github.com/filecoin-saturn/DEPRECATED-media)
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [orchestrator](https://github.com/filecoin-saturn/orchestrator)
    - Saturn's centralised network orchestrator.
    - It SHOULD be migrated.
- [ ] [L2-node](https://github.com/filecoin-saturn/L2-node)
    - Legacy repo for Saturn's l2 nodes which are no longer relevant.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [L1-dashboard](https://github.com/filecoin-saturn/L1-dashboard)
    - A dashboard UI for Filecoin Saturn's L1 node. Hosted at https://dashboard.saturn.tech.
    - It SHOULD be migrated.
- [ ] [metrics-dashboard](https://github.com/filecoin-saturn/metrics-dashboard)
    - Hosts the metric aggregation code for Saturn's Grafana.
    - It SHOULD be migrated.
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
    - It SHOULD be migrated.
- [ ] [roadmap](https://github.com/filecoin-saturn/roadmap)
    - Saturn's old project roadmap.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [contracts](https://github.com/filecoin-saturn/contracts)
    - Saturn's smart contract tooling and the cli to deploy payouts.
    - It SHOULD be migrated.
- [ ] [zk-fraud](https://github.com/filecoin-saturn/zk-fraud)
    - A tentative project to create a simple zk circuit which replicates the fraud detection algorithm on node logs
    - AFAICT this is not being used
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [payouts-website](https://github.com/filecoin-saturn/payouts-website)
    - The repo behind https://payouts.saturn.tech, the website where node operators interact with Saturn's payout smart contract to receive their FIL payouts for network contributions.
    - It SHOULD be migrated.
- [ ] [caboose](https://github.com/filecoin-saturn/caboose)
    - A remote-car blockstore which provides a blockstore interface over a dynamic set of remote car providers. This was heavily used during Project Rhea.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [ansible](https://github.com/filecoin-saturn/ansible))
    - The tools to provision, manage and configure Saturn trusted/core nodes.
    - It SHOULD be migrated.
- [ ] [explorer](https://github.com/filecoin-saturn/explorer)
    - A geospatial visualization for Saturn network stats. Live at https://explorer.saturn.tech.
    - It SHOULD be migrated.
- [ ] [saturn-analysis](https://github.com/filecoin-saturn/saturn-analysis)
    - Documents the fraud and payment analysis performed for the Saturn Network.
    - It SHOULD be migrated.
- [ ] [nginx-car-range](https://github.com/filecoin-saturn/nginx-car-range)
    - Nginx plugin for filtering range requests from CAR files.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
- [ ] [saturn-docs](https://github.com/filecoin-saturn/saturn-docs)
    - Saturn documentation hosted by Super at docs.saturn.tech.
    - It SHOULD be migrated.
- [ ] [rs-fevm-utils](https://github.com/filecoin-saturn/rs-fevm-utils)
    - Repo for fEVM related utility functions in rust which is a depdency of [contracts](https://github.com/filecoin-saturn/contracts).
    - It SHOULD be migrated.
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
    - It SHOULD be migrated.
- [ ] [saturn-demo](https://github.com/filecoin-saturn/saturn-demo)
    - A demo Github Pages website which uses the Saturn network and it's helpful to demo it.
    - It SHOULD be migrated.
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
    - It SHOULD be migrated.
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
    - It SHOULD be migrated.
- [ ] [brand-assets](https://github.com/filecoin-saturn/brand-assets)
    - WIP brand assets for the Saturn brand.
    - It SHOULD NOT be archived.
    - It SHOULD be migrated.
- [ ] [project-tracking](https://github.com/filecoin-saturn/project-tracking)
    - Project tracking for Saturn.
    - It SHOULD be archived.
    - It SHOULD be migrated.
- [ ] [project-tracking](https://github.com/filecoin-saturn/project-tracking)
    - Project tracking for Saturn.
    - It SHOULD be archived.
    - It SHOULD NOT be migrated.
    - [The project board](https://github.com/orgs/filecoin-saturn/projects/1/views/1) SHOULD be checked for anything important.
- [ ] [compare-payouts](https://github.com/filecoin-saturn/compare-payouts)
    - The companion tool to process [contracts](https://github.com/filecoin-saturn/contracts) payment CSVs with credits.
    - It SHOULD be migrated.

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
