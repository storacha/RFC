# Authorizing access to services

refs https://github.com/storacha/RFC/pull/67

## Problem

We've recently deployed UCAN authorized retrievals, which has triggered a number of realizations WRT authorizing access to services with UCANs.

For example, `space/index/add` causes the Indexing Service to fetch an index and add the hashes to it's IPNI chain (for internal network resolution). It is not authorized to fetch the index, so it can not fetch it!

This is just one of a number of authorizations needed within the network. It is not only authorized data retrievals that this problem affects. We authorize the upload service, indexing service and storage nodes to perform invocation on each other as well. The crux is that in order to execute a UCAN invocation, the issuer needs to prove they are authorized to perform the task. For that they need a delegation.

Authorizing services to talk to each other is happening more and more often as wel build out the network. Our current approach is to pre-authorize services, storing long lived delegations for use when needed. This is problematic because of a few reasons:

1. Adding additional capabilities requires a service re-deployment or additional orchestration to distribute new delegations. It can sometimes be a very manual and error prone process to ensure the right DIDs are used and the correct abilities are specified.
1. The only way to prevent access (for example in a security breach) is to use revocations, which is considered a last resort. Ideally UCAN delegations should be issued when needed and be short lived so that we do not need to rely on revocations as heavily as we currently do.
1. When we do not use pre-authorized delegations, it is cumbersome to attach additional required delegations to an invocation and extract them when the invocation is received.

## Proposal

### Authorize Access on Demand

The Upload Service already has a capability called `access/authorize`, which is used in the existing email-based authorization flow to request access to a space for an account (`did:mailto:`).

This RFC proposes we implement a very similar capability on Storage Nodes (Piri) and the Indexing Service.

You invoke `access/authorize` to obtain a delegation for the requested capabilities.

`access/authorize` is self signed, it does not need a delegation to invoke. It is up to the executor to decide whether they want to issue a delegation.

The caveats for `access/authorize` include the capability(s) requested (`can`, and optional `nb`) and an optional `cause`

### Add `blob/retrieve` "service" capability

I'm proposing `blob/retrieve` as a new capability that allows retrieving a blob in full. It will be used for internal operations like replication, repair, indexing and Filecoin onboarding - egress MUST NOT be recorded for these invocations.

Note that unlike `access/authorize` in the Upload Service, this `access/authorize` capability will not involve any attestations: a Storage Node will have the natural authority to `blob/retrieve` itself, and be able to delegate that to whomever it deems necessary, as normal in UCAN.

#### What can use this for?

##### Authorizing Storage Nodes to `blob/retrieve` on Storage Nodes

This is for replication purposes. The cause field will allow the recipient to validate the retrieval is for the purpose of replication. The cause should be the `blob/replica/allocate` invocation.

e.g.

```js
{
  iss: 'did:key:zStorageNodeAlice',
  aud: 'did:key:zStorageNodeBob',
  att: [{
    can: 'access/authorize',
    with: 'did:key:zStorageNodeAlice',
    nb: {
      // requested capabilities
      // Note: `with` is implied (did:key:zStorageNodeBob)
      att: [{
        can: 'blob/retrieve',
        nb: {
          // can be specific or open ended
        }
      }]
    }
  }],
  // invocation that caused the authorize request (attached to the invocation)
  // i.e. `blob/replica/allocate`
  cause: { "/": "bafy..." }
}
```

##### Authorizing the Upload Service to `blob/allocate`, `blob/accept`, `blob/replica/allocate` on Storage Nodes

Upload Service just needs the storage node DID and URL. It can optionally store a delegation until it expires.

Storage Nodes will be set up to issue the delegation by allow-listing `did:web:up.storacha.network`.

e.g.

```js
{
  iss: 'did:web:up.storacha.network',
  aud: 'did:key:zStorageNode',
  att: [{
    can: 'access/authorize',
    with: 'did:web:up.storacha.network',
    nb: {
      // requested capabilities
      // Note: `with` is implied (did:key:zStorageNode)
      att: [{
        can: 'blob/allocate',
        nb: {
          // can be specific or open ended
        }
      }, {
        can: 'blob/accept',
        nb: {
          // can be specific or open ended
        }
      }, {
        can: 'blob/replica/allocate',
        nb: {
          // can be specific or open ended
        }
      }],
      // invocation that caused the authorize request (attached to the invocation)
      // i.e. `space/blob/add`
      cause: { "/": "bafy..." }
    }
  }],
  exp: 12345 // now + 1 day or something?
}
```

##### Authorizing Roundabout to `blob/retrieve`🆕 on Storage Nodes

Roundabout is used by Filecoin SPs to fetch data for inclusion in Filecoin deals.

e.g.

```js
{
  iss: 'did:web:roundabout.web3.storage',
  aud: 'did:key:zStorageNode',
  att: [{
    can: 'access/authorize',
    with: 'did:web:roundabout.web3.storage',
    nb: {
      // requested capabilities
      // Note: `with` is implied (did:key:zStorageNode)
      att: [{
        can: 'blob/retrieve',
        nb: {
          // can be specific or open ended
        }
      }]
    }
  }]
}
```

##### Authorizing the Indexing Service to `blob/retrieve`🆕 on Storage Nodes

Typically the indexing service requires a delegation to retrieve indexes on behalf of the user, however when content is _first_ uploaded and an index added, the indexing service needs to retrieve the index, cache the hashes and add them to it's own IPNI chain. At this point a `blob/retrieve` invocation might be appropriate.

Storage Nodes will be set up to issue the delegation by allow-listing `did:web:indexer.storacha.network`.

e.g.

```js
{
  iss: 'did:web:indexer.storacha.network',
  aud: 'did:key:zStorageNode',
  att: [{
    can: 'access/authorize',
    with: 'did:web:indexer.storacha.network',
    nb: {
      // requested capabilities
      // Note: `with` is implied (did:key:zStorageNode)
      att: [{
        can: 'blob/retrieve',
        nb: {
          // can be specific or open ended
        }
      }],
      // invocation that caused the authorize request (attached to the invocation)
      // i.e. `assert/index`
      cause: { "/": "bafy..." }
    }
  }]
}
```
