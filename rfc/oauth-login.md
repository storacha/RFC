# RFC: OAuth Login

## Authors

- [Alan Shaw](https://github.com/alanshaw), [Storacha Network](https://storacha.network/)

## Introduction

There is currently only a single method for "login" to the Storacha network - Email. In short, the flow is such that a user receives an email, clicks on a link and the service creates a delegation from the email account (`did:mailto`) to the agent (`did:key`) attempting to login and includes a signed attestation that allows this. All delegations from spaces to email accounts (`did:mailto`) can therefore be used by any agent that successfully completes the email flow.

OAuth is a common method of authenticating users with a trusted third party service. Typcally users are directed to the third party website where they are prompted to accept the authorization and the scope of permissions that have been requested. On successful authentication, a callback URL is requested, indicating to the service trying to log the user in that the user successfully authorized themselves against the third party.

Oftentimes the third party already knows the user's email address and has already taken precautions to verify it. This RFC proposes a mechanism of authorizing a user against a third party service, allowing email verification to be skipped because the trusted third party has already performed this check.

## Proposal

If a trusted third party has already verified a user email address then skip verifying email address for a user.

When logging in with a trusted third party, login _does not_ involve prompting the user for an email address.

For example:

1. User requests login via Github
1. User authenticates via GitHub, their primary, verified email from GitHub is `user@example.com`
1. User is logged into Storacha as `user@example.com`

This is possible because we turst GitHub to verify email addresses, and will issue an attestation for this, provided GitHub indicates that they have previously verified `user@example.com`.

Note: this flow allows insertion into our existing email auth flow in a way that allows login with the same email via the non-third party flow possible.

### Implementation details

Typically clients call `access/authorize` specifying the capabilities they want. Currently the client knows the `did:mailto` of the issuer they want to receive the authorization from, since this is the email address the user is trying to login with. As such it is currently a _required_ parameter of the invocation (`nb.iss`). In the OAuth flow we do not know the email address beforehand, so the (backwards compatible) proposal is to make this field **optional**.

When authenticatiing against OAuth applications it is often possible to pass state from the client requesting authorization to the OAuth callback that is called on successful authentication.

We propose passing a signed `access/authorize` delegation as the state.

For example, a URL for authenticating with GitHub:

```js
const state = base64.encode((await accessAuthorizeDelegation.archive()).ok)
await fetch(`https://github.com/login/oauth/authorize?scope=read:user,user:email&client_id=XYZ&state=${state}`)
```

The OAuth callback URL might be `/oauth/github/callback`. The handler for which will unpack the `access/authorize` delegation, store it, and invoke `access/confirm`, using the primary, verified email address as the `nb.iss` field. As an expected side effect this will create an attestation and allow the delegation to be claimed via `access/claim`.

When claiming the delegation the client will create a local account using the `did:mailto` in the attestation.

## Appendix

The idea for this is to allow a _trial_ plan to be assumed by new logins. The OAuth callback will, before invoking `access/claim` create a customer in the system, and set their plan to `did:web:trial.storacha.network`.

After claiming their delegation the user will _not_ be presented with a payment wall because they already have a plan selected.
