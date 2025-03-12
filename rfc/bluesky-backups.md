# Bluesky Backups

## Goal

* A Bluesky user can OAuth our server into their Bluesky account and have the server back up to Storacha, on a schedule.
* As much state as possible is stored in the Storacha Space.

## Already Done

* A browser implementation, which stores metadata about uploaded backups in IndexedDB.
* A CLI implementation, which does not persist any metadata.

## Proposal

*This is the proposed destination. A breakdown of the iterative path to get here is below.*

The browser client, CLI client, and a new server app all share state in the Space via a Merkle clock. The new server app hosts that clock.

### Process Sketch

1. The user logs (their agent) into Storacha.
   * In addition to `space/*` capabilities, the agent claims `clock/*` capabilities on the Space.
2. The app fetches the clock head (`clock/head`) from the Service.
3. The user authorizes their app with Bluesky's OAuth.
   * *This is already built for the browser and CLI, but will also work on a new server version.*
4. The app fetches the latest repo and blobs. It then constructs a new Backup DAG containing those. It adds that to the Backups pail, resulting in a new head.
5. The app then instructs the Service to `clock/advance` the clock to the new state.
6. If this is the server app, the user may instruct the server to repeat 2-5 on a schedule, providing the necessary delegations.

### Data Sketch

* 🌐 **Bluesky Backups Service**
  * 🕰️ `w3clock`
    * 🔗 `head` of `did:key:zSpace` (→ 📄 Bluesky Backups)

* 🐔 **Storacha**
  * 📦 `did:key:zSpace`
    * 📄 Bluesky Backups (← 🔗 `head`)
      * 📄 `did:plc:ewvi7nxzyoun6zhxrhs64oiz`
        * 🪣 Backups
          * 📄 2025-03-12T15:13:59.958Z
            * 🦋 Repo
            * Blobs
              * 🖼️ bafyfullblobcid1
              * 🖼️ bafyfullblobcid2
              * …
          * 📄 2025-03-12T14:58:03.530Z
          * 📄 2025-03-12T13:51:36.993Z
          * …

#### Legend

* 🕰️ Clock service
* 📦 Space
* 📄 Generic IPLD-CBOR block
* 🦋 ATProto repository (root block, not CAR)
* 🖼️ Blob (UnixFS root block)

#### Notes

* `Blobs` is a map keyed by full-blob CIDs. These are the CIDs used in [blob links](https://atproto.com/specs/data-model#blob-type) in the repo. They point to UnixFS root blocks, as this is how the blob must be stored in Storacha.

### The Merkle Clock

* The Space is a clock: the same DID is used for both.
* A clock event is simply the new state. The model is largely insert-only, with (presumably) eventual deletion added to reclaim space, but never re-insertion. Thus, the state itself contains everything needed to implement a merge.
* Simultaneous clock events simply yield the union of the backups.
  * It's *theoretically* possible for two Agents to push two simultaneous insertions of backups with the exact same timestamp. If this happens, we should be able to safely throw one away, as they should be functionally identical, if not literally, bitwise identical.
* The clock service should validate that the clock is a provisioned Space. Otherwise, anyone could use it as a generic clock service for anything. (This is not at all critical for a demo—the only risk is becoming overloaded with operations and state that's not being paid for.)

## Iterative Path

We can get here in several smaller steps:

1. Browser app & CLI app store metadata in Storacha, and only store a ref CID in local storage.
2. Browser & CLI app track that ref as a `w3clock` Merkle clock.
3. Server app can do the same thing.
4. Server app can run on schedule.

## Open Questions

* Given the Merkle clock implemenation is relatively simple, is it overkill?