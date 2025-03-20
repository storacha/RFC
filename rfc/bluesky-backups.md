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
3. The app fetches the authorizations from the Space, and looks for a valid one for its DID/key.
4. If it doesn't already have authorization, the user authorizes their app with Bluesky's OAuth.
   * *This is already built for the browser and CLI, but will also work on a new server version.*
5. The app fetches the latest repo and blobs. It then constructs a new Backup DAG containing those. It adds that to the Backups pail, and adds its authorization (if not already there), resulting in a new head.
6. The app then instructs the Service to `clock/advance` the clock to the new state.
7. If this is the server app, the user may instruct the server to repeat 2-5 on a schedule, providing the necessary delegations.

### Data Sketch

* 🌐 **Bluesky Backups Service**
  * 🕰️ `w3clock`
    * 🔗 `head` of `did:key:zSpace` (→ 🗂️ Bluesky Backups)

* 🐔 **Storacha**
  * 📦 `did:key:zSpace`
    * 🗂️ Bluesky Backups (← 🔗 `head`)
      * 🗂️ `did:plc:ewvi7nxzyoun6zhxrhs64oiz`
        * 🗂️ Authorizations
          * 🗂️ `did:key:zAgent1`
            * Authorization info, encrypted with `did:key:zAgent1`'s key.
            * Expiration time
          * 🗂️ `did:key:zAgent2`
          * 🗂️ `did:key:zAgent3`
            * …
        * 🪣 Backups
          * 🗂️ `2025-03-12T15:13:59.958Z`
            * 🦋 Repo
            * Blobs
              * 🖼️ `bafyfullblobcid1`
              * 🖼️ `bafyfullblobcid2`
              * …
            * 📄 Preferences
            * ❔ Conversations
          * 🗂️ `2025-03-12T14:58:03.530Z`
          * 🗂️ `2025-03-12T13:51:36.993Z`
          * …

#### Legend

* 🕰️ Clock service
* 📦 Space
* 🗂️ Generic DAG-CBOR block
* 🦋 ATProto repository (root block, not CAR)
* 🖼️ Blob (UnixFS root block)
* 📄 JSON (text)
* ❔ Unknown (not specified in this RFC)

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

1. Server app can OAuth, store auth, and do backups just like browser & CLI apps. All auth and metadata is stored in a centralized relational database.
   * *Value: Once started, backups can run while customer's computer is offline.*
   * *Value: Customer doesn't need to worry about losing local browser storage.*
   * *Value: Customer can use the same backup set and auth from any browser.*
2. Server app can run on a schedule. (← [🚩 Bluesky Backups V1 Launch](https://github.com/storacha/project-tracking/milestone/1), April 01, 2025)
   * *Value: Customer knows their data is always backed up without taking any manual action.*
3. Each app produces a backup in the format specified above (the items within `🪣 Backups`). Auth and metadata are still stored in a centralized relational database, but metadata is simpler: it only tracks a single CID per backup. No Pail or Merkle clock are involved yet.
   * *Value: Customer depends less on central storage to hold data.*
4. Apps keep backups in a Pail, and the Pail and authorizations are stored in the Space and tracked by a`w3clock` Merkle clock. The service hosts the clock. No other information is stored centrally.
   * *Value: Customer no longer depends on central storage to hold data. Only the clock is centralized, and easily forked to move to another service if desired.*
   * *Value: Storacha has a stronger story for app development.*

## Open Questions

* Given the Merkle clock implemenation is relatively simple, is it overkill?
* Is there any real reason to keep authorization in the Space, or is that just silly?