# Curio API RFC

These are some suggested endpoints for collaboration between Curio and hot storage market software such as Storacha.

## Piece Storage & Retrieval

These endpoints are simply used for putting arbitrary blobs of bytes that has to a PieceCID of any valid size (i.e. 127/128 * 2^n where n is a positive whole number)

### GET /piece/{piece cid v2}
### HEAD /piece/{piece cid v2 }

These can just follow FRC-066: https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0066.md (please include the Range parameter!)

With the additional stipulation that they should probably ONLY accept v2 Piece CID: https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0069.md

### PUT /piece/{piece cid v2}

This is a new API for storing pieces of arbitrary size into Curio's storage. The request body should contain the bytes for the piece. These should be checked internally to verify they match the piece cid specified on put.

TBD: Should a header or query param specify that you intend to use this for PDP proofs later? (instructing curio to cache a proof tree)

Returns:
200/204 When piece is already there
201 When a new piece is created
400 when piece data does not hash to piece cid v2

## PDP Proving

### Approach #1 - Minimal, just prove within a piece

In this approach, we leave things very simple on curio's side, leave most lotus interaction to the market software and curio simply provides proofs within a piece.

This means that market software would need to:
1. Manage proof sets on chain (via lotus)
2. Monitor proving epochs
3. Communicate with DRAND
4. Produces challenges for a given seed
5. Translate challenge offsets within an overall proof set as a logical data array to challenge offsets within a single piece
6. Reassemble challenges against roots and submit the overall proof to the chain

### POST /prove

Request body:
```json
{
  "pieceCid": "~piece~cid~v2~previously~posted~",
  "offset": "~offset~in~piece~for~this~challenge~"
}
```

Response:
```json
{
  "merkleProof": { 
    // not sure what format this takes
  }
}
```

### Approach #2 -- Curio is a full PDP provider

In this scenario, almost all PDP work is managed by curio itself (which would then need alternate software when running without curio). Essentially curio would provide APIs that mirrored the Create/Add/Remove/Delete functions that will exist on chain for PDP proof sets, and then manage the submission of proofs on a schedule (I hear it's good at scheduling)

### POST /proof-sets

Request Body:
```json
{
  // should this be passed to curio? or is owner address built in to curio already?
  // should curio check it has signing capabilities for the address?
   "ownerAddress": "f3...",
   "challengePeriod": 15
}
```

Response:
Code: 201
Location header: "/proof-set/{set-id}"

*TODO: do we need an interim response given this is a chain transaction with a place to fetch the set-id later?*

### GET /proof-sets/{set-id}

Response:
Code: 200
Body:
```json
{
  "roots": [
    // Root ids in proof set in order
  ],
  "ownerAddress": "f3...",
  "challengePeriod": 15
}
```

### POST /proof-sets/{set-id}/roots

Append a root to the proof set

Request Body:
```json
{
  "pieceCid": "bafy....",
  "size": 1048576
}
```

This API should fail if the specified piece cid was not previously stored with the PieceCID API

Response:
Code: 201
Location Header: "/proof-sets/{set id}/roots/{root id}

### GET /proof-sets/{set id}/roots/{root id}

Response Body:
```json
{
  "pieceCid": "bafy....",
  "size": 1048576
}
```

### DEL /proof-sets/{set id}/roots/{root id}

Remove the given root id from the given proof set

### DEL /proof-sets/{set id}

Remove the specified proof set entirely

In this scenario, all submission of proofs is handled internally by Curio.
