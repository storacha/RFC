# RFC: UCAN Command Definitions
Status: Proposed Standard

## Introduction

This RFC proposes a definition and formal documentation format for UCAN Commands, to replace the manually created and inconsistently structured specification documents we currently use.

This is largely a proposal for UCAN itself, but as it's motivated by needs arising in Storacha, it will at least begin as a draft RFC here for initial feedback. It is fully compatible with the existing UCAN 1.0 specifications, and could be adopted by UCAN itself or live on top of it. However, its utility is limited if it is not adopted outside of a single organization.

### Notation

*Several elements of the proposal below will be illustrated with examples displayed in DAG-JSON. The canonical representations of such elements, however, is DAG-CBOR.*

*Comments such as `// "/": "bafy..` denote the CIDs of objects.*

## Problem

Understanding a UCAN document requires a great deal of implicit context. UCAN provides strict semantics for determining whether an invocation of a Command was issued with authority, and should therefore be executed. It does not, however, provide strict semantics for what that execution should consist of.

Consider the following scenario. Alice has some files in a home directory on a server. She'd like Bob to be able to access those files. Alice trusts Bob fully with the contents of this home directory, and so issues the following delegation:

```json5
{
  // "/": "bafy..accessdel",
  "iss": "did:example:alice",
  "aud": "did:example:bob",
  "sub": "did:example:alice",
  "cmd": "/home/access",
  "nonce": "…",
  "exp": null,
}
```

But Bob, whether out of malice or misunderstanding, then issues the following invocation:

```json5
{
  "iss": "did:example:bob",
  "aud": "did:example:alices-front-door",
  "sub": "did:example:alice",
  "cmd": "/home/access",
  "nonce": "…",
  "exp": null,
  "prf": [{"/": "bafy..accessdel"}]
}
```

`did:example:alices-front-door` is the smart lock on Alice's front door. Seeing that Bob has authority from Alice to "access" her "home", it accordingly unlocks and lets Bob in, where he has a very different kind of access than what Alice intended to delegate.

This is a kind of [confused deputy problem](https://en.wikipedia.org/wiki/Confused_deputy_problem). The issue lies in two related problems with UCAN Commands as so far defined in the UCAN specifications:

* UCAN Commands are not namespaced, so a Command name built from everyday words is likely to be ambiguously defined by unrelated specifications.
* UCAN Commands are not explicitly connected to any sort of documentation or specification, so any interpreter must correctly know the specification out-of-band.

[The UCAN specification states](https://github.com/ucan-wg/spec#33-command):

> Commands are concrete messages ("verbs") that MUST be unambiguously interpretable by the Subject of a UCAN. Commands are REQUIRED in invocations. Some examples include `/msg/send`, `/crud/read`, and `/ucan/revoke`.

However, while the *Subject* may unambiguously understand the Command, it is the *Executor* who must ultimately understand it, and the Executor may not know anything about the Subject other than that it has granted some authority to some other principal.

## Proposal

### Namespacing

The first part of the proposal is simple: a UCAN-based specification SHOULD namespace its Commands using a DID it controls. Thus, these may be two different Commands:

```json5
{
  // ...
  "cmd": "/did:web:ezhosting.example/home/access"
}

{
  // ...
  "cmd": "/did:web:smartlocksrus.example/home/access"
}
```

These DIDs both use the `web` method, and this is likely to be a common choice, but it is not required in this proposal. Any DID which the definer controls will do.

These Commands are valid under the base UCAN specification. They also parallel that specification's reservation of a `/ucan` namespace for Commands defined in UCAN specifications.

### Definition and Schema

Namespacing Commands resolves the ambiguity, but because it connects each Command with an authoritative Principal it also gives us the opportunity to specify and document Commands more clearly.

The following is a definition for a Command which publishes a blog post using a protocol defined by `did:example:blog`, a blogging service:

```json5
{
  // "/": "bafy..pubdef",
  "cmd": "/did:example:blog/post/publish",
  "description": "Publish a new blog post",
  "sub": "The blog where the post will be published",
  "args": {
    "article": {
      "type": "object",
      "properties": {
        "title": {
          "type": "string",
          "description": "The title of the article"
        },
        "author": {
          "type": "string",
          "description": "The DID of the article's author"
        },
        "content": {
          "type": "string",
          "description": "The text content of the article"
        }
      },
      "required": ["title", "author", "content"],
    },
  },
  "out": {
    "ok": {
      "type": "object",
      "properties": {
        "postId": { 
          "type": "string",
          "description": "The identifier of the published post",
        },
        "publishedAt": {
          "type": "string",
          "format": "date-time",
          "description": "The time and date when the post published."
        },
      },
      "required": ["postId", "publishedAt"],
    },
  },
}
```

This provides a number of useful pieces of information:

* The Command has a human-language definition (here, in English) for the action it will perform.
* The Command has a human-language definition for the role the Subject plays in the action.
* The arguments to the Command are fully specified and defined, using JSON Schema.
* The out-value expected in the Receipt is fully specified and defined, using JSON Schema.

This level of definition opens several new opportunities:

* Generic UCAN tools which can help a user manually create a Delegation or Invocation.
* Generic UCAN validation tools, either standalone or within UCAN libraries.
* Automatic, standard documentation generation.
* Static code generation of types in multiple programming languages.

### Assertion of Definition

The Command definition form given above may be used today to replace or supplement existing protocol documentation for existing Commands. Just as documentation today must be trusted by the reader according to out-of-band information (such as knowing that it's stored in a repo controlled by the same people who implemented the eventual Executor of the Command), these machine-readable definitions may also be asserted without cryptographic proof, and be useful.

However, if a Command is namespaced by a Principal's DID, as described above, that Principal can also meaningfully assert that a particular definition is correct. Such an assertion could look like this:

```json5
{
  "iss": "did:example:blog",
  "sub": "did:example:blog",
  "cmd": "/ucan/command/define",
  "args": {
    "def": {
      "/": "bafy..pubdef",
    },
  },
  "nonce": "…",
  "exp": null,
}
```

Because this assertion is signed by `did:example:blog`, it serves as proof that `bafy..pubdef` is the correct definition of the Command.

## Conclusion

The proposal offers three ideas:

* A convention of namespacing Commands with a DID controlled by the definer.
* A formal definition schema for Commands, including documentation.
* A Command for asserting the correctness of a particular definition of a Command controlled by the Subject.

The first two can be used independently. The third ties the first two together to make them more useful in a truly decentralized environment.

## Not yet addressed

* Sometimes the authority to invoke one command implies the authority to invoke another. This is partly handled by the Command name structure, as the authority to `/foo` implies the authority to `/foo/bar` and to `/foo/baz`. However, some desired authority derivation is more complex than this. Currently, Ucanto allows the server to define derivation logic, but it's up to each server to do so correctly. What's proposed already provides a signed document which could describe that logic in human language. However, it may be valuable to include a machine-readable definition of that logic which any compliant service could implement automatically.

* Sometimes we update a Command's Argument schema. Currently, we don't have a great solution for versioning our Commands/Abilities, and this proposal does nothing to improve that situation.

* This proposal makes no particular effort to allow for documentation in multiple human languages, as JSON Schema itself does not support multiple languages. However, there are proposals for JSON Schema which could be incorporated if adopted.

* Many technical details are currently elided, as they're unnecessary at this early stage. The definition assertion, in particular, is a very rough-draft proof of concept.