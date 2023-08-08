## Abstract

Customers using w3up from "serverless" functions must currently invoke and execute `access/claim` during
every invocation of their function, wasting valuable time and network resources. We propose a solution involving
the encoding of the `AgentDataExport` datastructure into a string suitable to be used as the value of an environment variable.


## Problem

More than one potential customer is integrating with w3up from a "serverless"
function (for example, from a [Next.js API Function](https://nextjs.org/learn/basics/api-routes)).
These integrations have not been straightforward, in large part because `w3up-client` is currently
designed to assume that clients are able to store delegations in durable storage (ie, 
[IndexedDB](https://github.com/web3-storage/w3up/blob/main/packages/access-client/src/drivers/indexeddb.js)
in the browser or [Conf](https://github.com/web3-storage/w3up/blob/main/packages/access-client/src/drivers/memory.js)
in the command line client). Serverless environments in general do not have natural access to durable storage like
this, and as a result current integrators are forced to "claim delegations" using [`access/claim`](https://github.com/web3-storage/specs/blob/main/w3-access.md#claim-access) on every invocation of the serverless function.

## Solution

We propose an algorithm for serializing the existing [`AgentDataExport`](https://github.com/web3-storage/w3up/blob/main/packages/access-client/src/types.ts#L141) datastructure
to a `string` that can be added as the value of an environment variable in, eg, [Next.js](https://nextjs.org/docs/pages/building-your-application/configuring/environment-variables) or
[Github Actions](https://docs.github.com/en/actions/learn-github-actions/variables) configuration. 

We also propose that the DAG House team implement a [`Driver`](https://github.com/web3-storage/w3up/blob/main/packages/access-client/src/drivers/types.ts#L4) that reads from a well-known (and configurable) environment variable
at initialization and prints an `AgentDataExport` encoded by the aforementioned algorithm to console on save. This will
give serverless function integrators an easy path to working with w3up.

### Encoding `AgentDataExport`

The `AgentDataExport` datastructure is defined by the following TypeScript type definitions:

[https://github.com/web3-storage/w3up/blob/main/packages/access-client/src/types.ts#L141](https://github.com/web3-storage/w3up/blob/main/packages/access-client/src/types.ts#L141)
```
export type AgentDataExport = Pick<
  AgentDataModel,
  'meta' | 'currentSpace' | 'spaces'
> & {
  principal: SignerArchive<DID, SigAlg>
  delegations: Map<
    CIDString,
    {
      meta: DelegationMeta
      delegation: Array<{ cid: CIDString; bytes: Uint8Array }>
    }
  >
}

...

export interface AgentDataModel {
  meta: AgentMeta
  principal: Signer<DID<'key'>>
  /** @deprecated */
  currentSpace?: DID<'key'>
  /** @deprecated */
  spaces: Map<DID, SpaceMeta>
  delegations: Map<CIDString, { meta: DelegationMeta; delegation: Delegation }>
}

...

export interface DelegationMeta {
  /**
   * Audience metadata to be easier to build UIs with human readable data
   * Normally used with delegations issued to third parties or other devices.
   */
  audience?: AgentMeta
}

...

export interface AgentMeta {
  name: string
  description?: string
  url?: URL
  image?: URL
  type: 'device' | 'app' | 'service'
}
```

We propose that this datastructure be serialized first to [CBOR](https://cbor.io/) and
the result be encoded as a [`base64pad` multibase string](https://github.com/multiformats/multibase).

Deserializers will need to be careful to deserialize the values of the `AgentDataModel` `delegations` and `spaces` 
to JavaScript `Map` types (!! does this happen automatically? !!).

### `EnvVarDriver` implementation

The `@web3-storage/access` library defines a `Driver` with the following TypeScript type definition:

```
export interface Driver<T> {
  /**
   * Open driver
   */
  open: () => Promise<void>
  /**
   * Clean up and close driver
   */
  close: () => Promise<void>
  /**
   * Persist data to the driver's backend
   */
  save: (data: T) => Promise<void>
  /**
   * Loads data from the driver's backend
   */
  load: () => Promise<T | undefined>
  /**
   * Clean all the data in the driver's backend
   */
  reset: () => Promise<void>
}
```

We will implement `EnvVarDriver` as a concrete type extension of `Driver`, ie,
`Driver<AgentDataModel>`, since the `load` method will need to be aware of the fact
that `spaces` and `delegations` should be instantiated as JavaScript `Map` types.

