# RFC: Consolidating `w3up` + `upload-service`

## Authors

- [Alan Shaw](https://github.com/alanshaw), [Storacha Network](https://storacha.network/)

## Introduction

For the event held in Bangkok, I forked `w3up` and `w3infra` and deployed a new service, and modules, with some changes to allow the w3up service to store data on decentralized storage nodes. I named the new repos `upload-service` and `upload-service-infra`.

We now need to plan a route forward to consolitate these services without breaking compatibility. This RFC is a proposal of that plan.

_Not in scope: this is not a proposal for monorepoing all the things. This is a proposal for how to remove the repo duplication in the existing arrangement of things._

### What is the difference?

1. JS modules published to `@storacha` namespace on npm with the same names. _Exception_: `@web3-storage/w3up-client` renamed to just `@storacha/client`.
1. `blob/allocate` and `blob/accept` are no longer invoked on the service, they are invoked on an external service i.e. storage nodes. _Note_: they have actually been _renamed_ from `web3.storage/blob/allocate` and `web3.storage/blob/allocate`.
1. Removed the Store Protocol related code...it just doesn't work in a decentralized context and is deprecated anyway. _Exception_: I left the `store/*` capabilities defined in `@storacha/capabilities` (and the `web3.storage/*` ones for that matter).
1. Merged `w3cli` into the monorepo as `packages/cli` (and published on npm as `@storacha/cli`).
1. Deployed to `https://upload.storacha.network` (yes, unfortunately `https://up.storacha.network` is already taken), with service DID `did:web:upload.storacha.network`. _Note_: new service DID includes `upload.` i.e. it is NOT `did:web:storacha.network`.
1. Removed `allocations` table, added `blob-registry` table. Instead of recording allocations, we now "register" blobs. Registration is performed _after_ a blob is **accept**ed by a storage node. This gives us a more accurate view of what is stored in a space - mitigating the issue of a client failing to upload a blob after requesting to store it and still being billed for the used space. The `space/blob/list` invocation uses this table.
1. There is no Filecoin pipeline deployed and many `w3infra` stacks relating to legacy infra have been excluded from `upload-service-infra`.

I believe these changes to be mainly _good_ for the project as a whole and aligned with where we want to get to but please leave comments to discuss if you have disagreement.

### Why did you do this?

* Easier, quicker and safer to fork and deploy a greenfield project (do not have to worry about affecting existing production deployment).
* Time pressure necessitated it.
* Fresh start, was able to shed some legacy baggage.
* As mentioned above, the changes are things we need to do eventually anyway IMHO.
* I wanted an as realistic as possible demo for our event in Bangkok.

## Proposal

We transition to working on `upload-service`, porting any changes made in `w3up` that do not yet exist in `upload-service`.

We _backport_ the relevant infra changes in `upload-service-infra` that were added to support "Routing Blob Protocol" to `w3infra`.

* Change the `w3infra` service DID to `did:web:upload.storacha.network` and URL to `https://upload.storacha.network`.
* Point `https://up.web3.storage` and `https://up.storacha.network` to `https://upload.storacha.network`.
* Allow the service to receive invocations to audience `did:web:web3.storage` and accept attestations signed by `did:web:web3.storage`.
* Add the legacy Store Protocol handlers to `upload-service` by importing existing `@web3-storage/upload-api`.
* Setup and deploy a production storage provider node that writes to carpark.
* Migrate `allocation` table data to `blob-registry` table - this allows `space/blob/list` capability to work for older spaces.
    * This is a table rename and a couple of field renames (`multihash` -> `digest` and `invocation` -> `cause`). For reference, I've listed the current schemas:
        * `allocation` schema:
            * `space`: string (e.g. `did:key:space`)
            * `multihash`: string (e.g. `zQm...`)
            * `size`: number (e.g. `101`)
            * `invocation`: string (e.g. `baf...ucan` - CID of invocation UCAN)
            * `insertedAt`: string (e.g. `2022-12-24T...`)
        * `blob-registry` schema:
            * `space`: string
            * `digest`: string
            * `size`: number
            * `cause`: string
            * `insertedAt`: string
    * We will create the new table and _temporarily_ write to both.
    * Once deployed we'll start the migration of older data to the new table.
    * Once the migration is completed we'll remove the old table.
* In order to prevent confusion over what modules we're continuing to support, we will remove from the repos as much legacy code as possible.
    * Import and re-export `@storacha` equivalents of all modules. Instead of simply deleting the packages this enables us to continue to easily make and track releases (bugfixes etc.) for the legacy packages whilst still removing the majority of the code.
    * `w3up` repo changes:
        * `access-client`
            * import `@storacha/access` and re-export
        * `blob-index`
            * import `@storacha/blob-index` and re-export
        * `capabilities`
            * import `@storacha/capabilities` and re-export
        * `did-mailto`
            * import `@storacha/did-mailto` and re-export
        * `eslint-config-w3up`
            * leave as is (can remove when monorepo'd)
        * `filecoin-api`
            * import `@storacha/filecoin-api` and re-export
        * `filecoin-client`
            * import `@storacha/filecoin-client` and re-export
        * `upload-api`
            * Leave Store and Blob Protocol related handlers as is.
            * Import `@storacha/upload-api` for all other handlers. This will remove most code from this package.
        * `upload-client`
            * import `@storacha/upload-client` and re-export
        * `w3up-client`
            * Import `@storacha/client` (and any other `@storacha/*` modules needed) and augment client with existing (but deprecated) Store Protocol methods.
    * `w3infra` repo changes:
        * import `@storacha/upload-api`
    * `w3cli` repo changes:
        * import `@storacha/cli` and re-export
            * ...but first make `@storacha/cli` usable as library
