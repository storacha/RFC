# Curio API RFC

This is a proposed set of APIs to be implemented in Curio to interface with a storacha node.

The proposed flow has three core API endpoints:
1. Endpoints for manipulating proofs sets
2. An endpoint for storing pieces + flow for storing pieces
3. An endpoint for retrieving pieces

There are addition considerations we should consider:
1. Authorization -- In storacha's network, it's important that the original end user maintain control of authorization for any action performed (including retrieval). We accomplish this through UCANs. We should discuss how we can maintain this without forcing curio to implement a full UCAN authorization process.
2. Aggregation - storacha's data is at times extremely small (<1mb in certain cases). Our understanding is that economically, it makes more sense to do some light aggregation of data before adding it to the proof set. The proposal below outlines a facility for doing this. While storacha would store pieces as it receives them, we would add them to the proof set in a seperate step, with a root that could optionally be an aggregate of several pieces.
3. IPNI announcements -- we plan to use IPNI announcements in a specific way with our pieces. Our understanding is the curio IPNI flow is in flux. We can try to integrate your IPNI api or just do it ourselves.

## Basic flow

In the proposal below, the basic flow is as follows;
1. Create a proof set for Storacha on the SP (happens just once)
2. Upload pieces from storacha with the piece storage API
   - at this point, the piece is immediately retrievable but not being proven
3. When enough pieces are received (128MB or more) create an aggregate root and add it to the proof set
   - at this point, all pieces submitted to the proof set are retrievable AND proven

## Proof sets API

The following API describes how to create, read, update and delete proofsets managed by Curion. Essentially curio ill provide APIs that mirrored the Create/Add/Remove/Delete functions that will exist on chain for PDP proof sets, and then manage the submission of proofs on a schedule (I hear it's good at scheduling)

### POST /proof-sets

Create a new proof set for a specific 
Request Body:
```json
{
  // need to drill down on these propoerties
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

Append a root to the proof set, which may be an aggregation of one or more piece cids.

Request Body:
```json
{
  "rootCid": "bafy....root",
  "pieces": [
    {
      "cid": "bafy...piece1",
      "proof": [
        "bafy...intermediate1",
        "bafy...intermediate2"
      ]
    },
    {
      "cid": "bafy...piece2",
      "proof": [
        "bafy...intermediate3",
        "bafy...intermediate4"
      ]
    },
    //...
  ],
  "size": 1048576
}
```

This API should fail if the all pieces were not previously stored with the Piece Storage API

Response:
Code: 201
Location Header: "/proof-sets/{set id}/roots/{root id}

### GET /proof-sets/{set id}/roots/{root id}

Response Body:
```json
{
  "rootCid": "bafy....root",
  "pieces": [
    {
      "cid": "bafy...piece1",
      "proof": [
        "bafy...intermediate1",
        "bafy...intermediate2"
      ]
    },
    {
      "cid": "bafy...piece2",
      "proof": [
        "bafy...intermediate3",
        "bafy...intermediate4"
      ]
    },
    //...
  ],
  "size": 1048576
}
```

### DEL /proof-sets/{set id}/roots/{root id}

Remove the given root id from the given proof set

### DEL /proof-sets/{set id}

Remove the specified proof set entirely


## Piece Storage


### POST /piece

This is a new API for storing pieces of arbitrary size into Curio's storage. 
```json
{
  "pieceCid": "{piece cid v2}",
  "notify": "optional http webhook to call once the data is uploaded"
}
```

Returns:
204 When piece is already there
201 When a new piece upload is created
the response should contain a location header with a URL that should be used for the actual upload. This URL should accept a PUT request with the actual bytes of the piece. The request should fail if the bytes do not hash to the correct piece CID.

## Piece Retrieval

These endpoints are simply used for retrieving blobs of any valid size

### GET /piece/{piece cid v2}
### HEAD /piece/{piece cid v2 }

These can just follow FRC-066: https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0066.md (please include the Range parameter!)

With the additional stipulation that they should probably ONLY accept v2 Piece CID: https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0069.md
