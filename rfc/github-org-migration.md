# Github Org Migration

![status:draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

## Editors

- [João Andrade](https://github.com/joaosa), [Protocol Labs](https://protocol.ai/)

## Authors

- [João Andrade](https://github.com/joaosa), [Protocol Labs](https://protocol.ai/)

## Abstract

Formalize a process for merging the [Filecoin Saturn](https://github.com/filecoin-saturn) and the [Web3 Storage](https://github.com/web3-storage) orgs into [w3s](https://github.com/w3s-project).

Ensure a common understanding on how repositories should be migrated on a case-by-case basis.

Provides recommendations and fixes to make this migration process as smooth and bulletproof for everyone.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

Migration is defined as the act of transferring ownership from either the [Filecoin Saturn](https://github.com/filecoin-saturn) or [Web3 Storage](https://github.com/web3-storage) orgs into [w3s](https://github.com/w3s-project) and ensuring all dependent systems continue operating transparently. If such a goal is not possible, then the necessary changes to either prevent the system from stopping its operation or recover it will be proposed.

Local Migration is defined as the act of moving a repo into the post-migration filesystem path while ensuring its [Git remote](https://git-scm.com/docs/git-remote) matches a path in the [w3s](https://github.com/w3s-project) org.

## Out of scope

- Tidying up local Git state post-migration (i.e. removing old remotes, and branches);
- TBD

## Migration consequences and Failure scenarios

According to [Transferring a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/transferring-a-repository): 

> "When you transfer a repository, its issues, pull requests, wiki, stars, and watchers are also transferred. If the transferred repository contains webhooks, services, secrets, or deploy keys, they will remain associated after the transfer is complete. Git information about commits, including contributions, is preserved.".
> (...)
> "All links to the previous repository location are automatically redirected to the new location. When you use git clone, git fetch, or git push on a transferred repository, these commands will redirect to the new repository location or URL. However, to avoid confusion, we strongly recommend updating any existing local clones to point to the new repository URL."

This ensures migration disruptions are minimal and that we can keep operating normally and migrate on a repo-by-repo basis. Still, it is crucial to look at each repo individually to ascertain migration consequences which may impact other systems outside of Github and our local development machines.
Additionally, it is RECOMMENDED to perform the [post-migration instructions](#post-migration-instructions) below.

CI pipelines SHALL be individually checked to make sure nothing breaks or relies on a particular repo naming structure.

## Repository Overview

**Note: This section is under construction and it is where the bulk of work of this spec will go**

We have a total of 60 candidate repositories to migrate.

The following repos lists were computed with: `curl -Ls -H "Accept: application/vnd.github+json" -H "Authorization: Bearer $GITHUB_TOKEN" -H "X-GitHub-Api-Version: 2022-11-28" https://api.github.com/orgs/$ORG/repos | jq -r '.[] | "- [ ] [" + .name + "](" + .html_url + ")"' | pbcopy`, where `$ORG ∈ {"web3-storage", "filecoin-saturn"}` and where `GITHUB_TOKEN` is a token with access to both orgs. Further reference [here](https://docs.github.com/en/rest/repos/repos?apiVersion=2022-11-28#list-organization-repositories).

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
- [ ] [ucanto](https://github.com/web3-storage/ucanto)
- [ ] [w3up](https://github.com/web3-storage/w3up)
- [ ] [ucanto-name-system](https://github.com/web3-storage/ucanto-name-system)
- [ ] [alt-gateway](https://github.com/web3-storage/alt-gateway)
- [ ] [link.web3.storage](https://github.com/web3-storage/link.web3.storage)

### [Filecoin Saturn](https://github.com/filecoin-saturn)

- [x] [DEPRECATED-saturn](https://github.com/filecoin-saturn/DEPRECATED-saturn)

An historical artefact of an initial Filecoin Saturn implementation. It SHOULD NOT be migrated.

- [ ] [browser-client](https://github.com/filecoin-saturn/browser-client)
- [ ] [L1-node](https://github.com/filecoin-saturn/L1-node)
- [ ] [homepage](https://github.com/filecoin-saturn/homepage)
- [ ] [terraform](https://github.com/filecoin-saturn/terraform)
- [ ] [lambdas](https://github.com/filecoin-saturn/lambdas)
- [ ] [DEPRECATED-media](https://github.com/filecoin-saturn/DEPRECATED-media)
- [ ] [orchestrator](https://github.com/filecoin-saturn/orchestrator)
- [ ] [L2-node](https://github.com/filecoin-saturn/L2-node)
- [ ] [L1-dashboard](https://github.com/filecoin-saturn/L1-dashboard)
- [ ] [metrics-dashboard](https://github.com/filecoin-saturn/metrics-dashboard)
- [ ] [http-testing](https://github.com/filecoin-saturn/http-testing)
- [ ] [kubo](https://github.com/filecoin-saturn/kubo)
- [ ] [js-client](https://github.com/filecoin-saturn/js-client)
- [ ] [roadmap](https://github.com/filecoin-saturn/roadmap)
- [ ] [contracts](https://github.com/filecoin-saturn/contracts)
- [ ] [zk-fraud](https://github.com/filecoin-saturn/zk-fraud)
- [ ] [payouts-website](https://github.com/filecoin-saturn/payouts-website)
- [ ] [caboose](https://github.com/filecoin-saturn/caboose)
- [ ] [ansible](https://github.com/filecoin-saturn/ansible)
- [ ] [explorer](https://github.com/filecoin-saturn/explorer)
- [ ] [saturn-analysis](https://github.com/filecoin-saturn/saturn-analysis)
- [ ] [nginx-car-range](https://github.com/filecoin-saturn/nginx-car-range)
- [ ] [saturn-docs](https://github.com/filecoin-saturn/saturn-docs)
- [ ] [rs-fevm-utils](https://github.com/filecoin-saturn/rs-fevm-utils)
- [ ] [fevm-deployment-tutorial](https://github.com/filecoin-saturn/fevm-deployment-tutorial)
- [ ] [L1-replay](https://github.com/filecoin-saturn/L1-replay)
- [ ] [L1-replay-go](https://github.com/filecoin-saturn/L1-replay-go)
- [ ] [size-diff-tool](https://github.com/filecoin-saturn/size-diff-tool)
- [ ] [misc](https://github.com/filecoin-saturn/misc)


## Post-migration instructions

### Local Migration

The goal of this section is to provide a concise and simple way to operate on a given repo post-migration without losing any local changes. 
This is provided in the form of shell commands (more specifically using `bash` and [GNU findutils](https://www.gnu.org/software/findutils)).

Exact repo path are provided as examples and may not result in a precise command for everyone. To account for this, `$PATH_PREFIX` should equal the path prefix that is required to reach a given repo from your current directory. For instance, while writing this, this means `PATH_PREFIX=~/ghq/github.com`.

A given `ORG` is assumed to exist in the one's filesystem beforehand. If not a [git clone](https://git-scm.com/docs/git-clone) targeting `https://github.com/w3s-project/$REPO` is RECOMMENDED (see the [Repository Overview](#repository-overview) for specific values for `REPO`).

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
