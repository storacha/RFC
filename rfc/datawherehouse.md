# DATAWHEREHOUSE

> or... where is that file living on?

## Authors

- [Vasco Santos], [Protocol Labs]

## Background

TL;DR

1. We're missing a mapping of Data CID -> target URI (Bucket, Saturn Node, etc).
2. We don't want clients to create location claims for internal bucket URLs because we might change the location in the future (reputation hit for client).
3. We don't have bucket events in Cloudflare, so need the client to tell us when it has uploaded something to the provided write target.
4. We want freeway code to be usable in Saturn nodes, so ideally it uses only content claims to discover locations.
5. We want this information available as soon as content is written so that read interfaces can serve the content right away.

### Extended details

> 1. We're missing a mapping of Data CID -> target URI (Bucket, Saturn Node, etc).

We considered at first location claims would be the solution here. But when we got closer to put that into practise we realized this was not a good idea. We need this mapping so that w3s Read/Write Interfaces can discover where the bytes are. Where the bytes are stored may actually be private write targets (for instance, a R2 bucket), which location is not public. We consider that locations claims MUST be retrievable, have public access and not heavily rate limited. Finally, some read interfaces (for instance Roundabout, Freeway) require some information encoded in the URI (like bucket name), which would not be available in a public URL of R2 bucket. All things considered, Location claims should include URIs like `https://bafy...data.ipfs.w3s.link`, and we need a mapping of `bafy...data` to where its bytes are actually stored internally.

> 2. We don't want clients to create location claims for internal bucket URLs because we might change the location in the future (reputation hit for client).

Extending on the first point, making location claims to include "private/unavailable" URIs will make it harder for the service to move data to other places, given it would need to revoke a bunch of claims and re-write them with new location.

> 3. We don't have bucket events in Cloudflare, so need the client to tell us when it has uploaded something to the provided write target.

Actually we can even extend on this point by saying that today we have no verifiability on data being sent by the user, as well as received by the service. By having the client to sign that bytes were sent, and the service to check and also sign that is true will allow us to achieve that. Moreover, we also open the doors in this interaction for a challengen/proof of delivery.

## High level flow of proposed solution

* Simulate bucket event by having client submit `store/deliver` invocation.
    * Confirms successful transfer of data.
    * Effect linked by `store/add` receipt.
    * Can be batched with other invocations like `filecoin/offer`
* Handler writes to carwhere
    * CAR CID -> bucket mapping table 
* Materialize location claims on demand for CAR CID from data in table. Short lived so we can change location and not have to revoke 1,000's of UCANs.

The following diagram presents the described flow, and is illustrated with the following steps:

1. Client requests to store some bytes with the storefront service
2. Service issues a receipt stating the URI of a write target for the client to write those bytes
3. Client writes the bytes into the write target
4. Client notifies the service that the bytes were written to the provided write target
5. Service verifies that the bytes are stored in the provided write target
6. Service issues a receipt stating bytes are being stored by the service.

![datawherehouse1](./datawherehouse/datawherehouse-1.svg)

On the other side of things, the next diagram presents the flow when a client wants to read some data, and is illustrated with the following steps:

1. Client request to read some bytes by the CID of the data
2. Service discovers where the requested bytes are stored relying on content claims service and the materialized claims from `datawherehouse`
3. Service serves data stored on the discovered location.

![datawherehouse2](./datawherehouse/datawherehouse-2.svg)

## carwhere store design

carwhere is a store that enables a `store/*` implementer to map CAR CIDs to one or more locations where they are written (and confirmed!).

The store should be indexed and queryable by CAR CID, but should also support multiple entries with CAR CID. Therefore, we have two potential solutions for this Store:
- Bucket store with format as `${carCid}/${bucketName}/${region}/${key}`. So, quite similar to current Dudewhere indexes. E.g. `bag.../carpark-prod-0/auto/bag.../bag....key`
- DynamoDB store where partition key is `${carCid}` and `${bucketName}/${region}/${key}`

From a price standpoint, as well as ease of storage migration, Bucket store will be way cheaper. However, dynamoDB will be faster for high throughputs. Given the index read will likely be one of the less costly parts of reading content, it MAY not make a big difference to read from the indexes. Specially as we will have full content cached later on.

Proposal: Bucket Store

## Bucket data Location URIs

Defining the format of data locations for these target locations is critical to have a mapping of these locations to the buckets to fulfill all requirements of read interfaces (See https://hackmd.io/5qyJwDORTc6B-nqZSmzmWQ#Read-Use-cases). 

### URIs in well known write targets

Typically, objects in S3 buckets can be located via following URIs:
- S3 URI (e.g. `s3://<BUCKET_NAME>/<CID>/<CID>.car`)
- Object URL (e.g. `https://<BUCKET_NAME>.s3.<AWS_REGION>.amazonaws.com/<CID>/<CID>.car`)
  - can be used to fetch the bytes by any HTTP client if bucket is public

However, R2 object locations have different patterns, instead of following S3 pattern. They can be:
- Public Object URL for Dev (e.g. `https://pub-<INTERNAL_R2_BUCKET_IDENTIFIER>.r2.dev/<CID>/<CID>.car`)
  - can be used to fetch the bytes by any HTTP client, if bucket is public and heavily rate limited 
      - [R2 docs](https://developers.cloudflare.com/r2/buckets/public-buckets/#enable-managed-public-access) state that such URLs should only be used for dev!
- Custom domain object URL (e.g. `https://<CUSTOM_DOMAIN>.web3.storage/<CID>/<CID>.car`)
  - can be used to fetch the bytes by any HTTP client, if custom domain is configured in R2
      - can be rate limited by operator configuration
      - account will need to pay for the egress of reading from the bucket
- Presigned URL (e.g. `https://<ACCOUNT_ID>.r2.cloudflarestorage.com/<BUCKET_NAME>/<CID>/<CID>.car?Signature=...&Expires=...`)
  - can be used to fetch  the bytes by any HTTP client, if has the signature query parameter and is not expired
  - no heavy rate limits in place, together with no egress costs to read data at rest

Note that a data location URI may not be readable from all actors, as some may be behind a given set of permissions/capabilities.

### URI Patterns

The main pattern that we can identify is to have URLs that can be accessed by any HTTP client. Except for S3 URIs and given the correct setup/keys is available, all other URLs are fetch'able. Therefore, we can assume as an advantage that claim is directly fetchable without any pre-knowledge.

Having a claim that cannot be used by every potential client (i.e. needs some extra permissions) is also a disadvantage that may represent penalties in a reputation system. Moreover, rate limits can have a negative impact on reputation as well.

URIs that include all necessary information to enable derivation for smart clients that could rely on Worker R2 Bindings or to generate Presigned URLs are critical for several use cases. URIs that minimize Egress costs can also be preferred by smart clients.

Per the above, considering S3, it looks like we should rely on S3 Object URL (e.g. `https://<BUCKET_NAME>.s3.<AWS_REGION>.amazonaws.com/<CID>/<CID>.car`).

But with R2, there is no perfect fit. The only good option would be the format used by presigned URLs, but it should only be created on request given their expiration. The custom domain offers better retrievability than the Public Object URL for Dev, but has not enough information encoded for smart clients (i.e. no way to know bucket name). For R2, we will likely need to add 2 location URIs:
- Custom domain object URL (e.g. `https://<CUSTOM_DOMAIN>.web3.storage/<CID>/<CID>.car`)
  - can be used out of the box to fetch content
- Presigned URL like URI without query params (e.g. `https://<ACCOUNT_ID>.r2.cloudflarestorage.com/<BUCKET_NAME>/<CID>/<CID>`)
  - won't really work, but smart clients can see if it is available and rely on its encoded info to use CF Worker Bindings / create presigned URLs, etc.

Alternatively, we can require `<CUSTOM_DOMAIN>` to become bucket name, in order to make it work as a single location URI. Main disadvantage besides the extra requirement, is that there is no url mention of being a R2 bucket, which would mean hard coded assumptions. We could also consider to mimic S3 URL for R2 here as well.

The Public Object URL for Dev should not be adopted, as it is heavily rate limited, we do not know what CF may do with it in the future, and also does not even have information about the Bucket name.

### Proposal

Nothing prevents us from claiming multiple location URIs for a given content, however we may need to also be careful on having multiple claims for the same location as if it is not fully available it MAY be ranked badly in whatever reputation system we may create. However, some smart clients MAY benefit of cost savings or faster retrievals if they have extra information encoded in the URI.

In conclusion, this document proposes that w3up clients, once they successfully perform an upload, they create location claims within following formats:
- S3
    - `https://<BUCKET_NAME>.s3.<AWS_REGION>.amazonaws.com/<CID>/<CID>.car`
- R2
    - `https://<CUSTOM_DOMAIN>.web3.storage/<CID>/<CID>.car`
    - `https://<ACCOUNT_ID>.r2.cloudflarestorage.com/<BUCKET_NAME>/<CID>/<CID>`

## Materialize location claims

Extend materialized location claims to include carwhere locations short lived. We will need to align on how these claims will look like.
