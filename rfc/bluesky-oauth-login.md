# RFC: Bluesky OAuth Ligin

## Authors

- [Travis Vachon](https://github.com/travis), [Storacha Network](https://storacha.network)

## Introduction

In order to upload data to the Storacha Network, users must first a) authenticate
and b) obtain a "storage plan." Originally, (b) required using the Stripe checkout
process to provide credit card information that could be used to charge for monthly
storage and overages, but in Q1 2025 we made it possible for users to sign up using
[GitHub OAuth](./oauth-login.md) which would simultaneously authenticate them (because
GitHub provides OAuth API access to a "verified" email address) and grant them
a basic storage plan with a 100MB data storage allocation. This helped decrease
the amount of friction in our signup process and provided a neat way to let people
use Storacha at hackathons and similar events - both big wins for a critical user
acquisition funnel for our service.

In this RFC we propose extending this pattern to the Bluesky OAuth flow - 
[recent additions to Bluesky OAuth](https://docs.bsky.app/blog/oauth-improvements#email-transitional-scope)
mean that we determine when a Bluesky's email address has been verified by the PDS
and, assuming we trust the PDS, authenticate a user and grant them a basic storage
allocation the same way we do with GitHub.

## Proposal

We propose implementing [OAuth Login](./oauth-login.md) for Bluesky OAuth. We'd like to
add a new OAuth endpoint using the same techniques that we do for GitHub today. The
primary difference between the existing GitHub OAuth callback and the new Bluesky OAuth 
callback is that we'll use the Bluesky API client rather than the GitHub API client to 
register and record OAuth access tokens. 

Importantly, we will follow the same pattern proposed in [OAuth Login](./oauth-login.md) 
to give users a small storage allocation if their Bluesky email is "verified".

An existing implementation of a Bluesky OAuth
callback, in use by our bsky.storage application, can be found here: 

https://github.com/storacha/bluesky-backup-webapp-server/blob/main/src/app/atproto/callback/page.tsx

Additional infrastructure and changes to support this are considered below.

### Session and State Stores

The ATProto (ie, Bluesky) OAuth client requires the implementation of two data stores:
one for OAuth state, used during OAuth signup, and one for OAuth sessions, used by our 
backend to execute backups. An existing implementation of these stores using PostgreSQL 
can be found here:

https://github.com/storacha/bluesky-backup-webapp-server/blob/5af6352bb4ac07e817975e3af43ebfed45fa7c8b/src/lib/server/db.ts#L98

For this project we will reimplement these in DynamoDB - the shape of this API is a better match for DynamoDB's semantics than PostgreSQL so this should be straightforward.

### Request Lock

Some operations provided by the ATProto OAuth client require a semaphore to avoid race conditions when interacting with the remote OAuth service. bsky.storage uses a SQL implementation of this interface that can be found here:

https://github.com/storacha/bluesky-backup-webapp-server/blob/5af6352bb4ac07e817975e3af43ebfed45fa7c8b/src/lib/server/db.ts#L183

And implementation of this interface built on [`redlock`](https://www.npmjs.com/package/redlock) can be found in the documentation for the ATProto OAuth client library:

https://www.npmjs.com/package/@atproto/oauth-client-node

We propose using this on top of one of AWS's Redis-like services.

### Client Metadata

The Bluesky OAuth process requires our server to expose two new endpoints to ensure
secure communication. The first exposes metadata about our OAuth implementation. An 
implementation of this can be found here:

https://github.com/storacha/bluesky-backup-webapp-server/blob/5af6352bb4ac07e817975e3af43ebfed45fa7c8b/src/app/atproto/oauth-client-metadata/%5Baccount%5D/route.ts

Note that passing `account` to the metadata function is not strictly required, and we expect to be able to get rid of this requirement in the new implementation.

### JWKS

Similar to `Client Metadata` above, the server must expose an endpoint that lists its JWK
public keys to facilitate secure communication between our server and the OAuth server.
The existing implementation can be found here:

https://github.com/storacha/bluesky-backup-webapp-server/blob/5af6352bb4ac07e817975e3af43ebfed45fa7c8b/src/app/atproto/oauth/jwks/route.ts


### Access to Access Tokens

The existing bsky.storage service needs access to at least the session store (and
possible the state store) in order to back up user data. There are two main options for
this:

1) We could add additional UCAN invocations that let bsky.storage get the data it needs from our backends
2) We could give bsky.storage access to the DynamoDB tables it needs

While (1) is probably more "correct" and in line with the way we build systems, (2) is
probably significantly less work and aligns with this being an internal application
rather than a third party integration. We therefore propose going with (2) which will
require adjusting the bsky.storage Storoku configuration to give that backend service
access to the DynamoDB tables it needs.

## Work Plan

1) Finish implementation of OAuth callback
2) Finish implementation of supporting endpoints (JWKs, client metadata)
3) Implement session and state stores in terms of Dynamo
4) Implement Request Lock using an AWS Redis-like service
5) Update bsky.storage's deployment configuration to give it access to the necessary Dynamo tables
6) Update bsky.storage to use the Dynamo-backed implementations of the session and state stores

## Draft Implementation

We have used Claude to create a rough sketch of the new OAuth callback - it is neither
bug-free nor complete, but should help give more context and detail to the proposal 
above:

https://github.com/storacha/w3infra/pull/505