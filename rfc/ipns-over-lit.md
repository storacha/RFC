# RFC: IPNS over Lit

## Authors

- [Alan Shaw](https://github.com/alanshaw), [Storacha Network](https://storacha.network/)

## Introduction

IPNS is a system for mutable references. In short, a user generates a keypair and signs "records" with the private key to change the current value. The value is accessed via the public key. Typically new records are published to the IPFS Amino DHT and distributed over libp2p gossipsub.

IPNS does not make provision for multi-writer unless the private key is shared. Only the holder(s) of the private key can issue updates to the value.

This RFC proposes a simple solution that enables multi-writer IPNS by leveraging the Lit protocol and UCAN.

## Proposal

Lit is a decentralized key management protocol. Use Lit to store the IPNS private key and invoke Lit to sign an IPNS record.

How do we authenticate requests? UCAN.

### Creating an IPNS key

A Lit Action takes an agent DID, generates a random private key and returns a signed UCAN delegating `name/increment` ("can") on the generated public key _resource_ ("with") to the agent DID ("audience").

Access to mutate the IPNS record can be delegated via UCAN to other agents.

### Mutating the IPNS record

Another Lit Action receives a UCAN invocation for `name/increment` with an IPNS record revision and delegation proof. The Lit Action validates the invocation using [ucanto](https://github.com/storacha/ucanto), signs and returns the IPNS record.

We can use the [w3name client](https://github.com/storacha/w3name#js-client) to construct IPNS records and ucanto to send invocations.

### Publishing and resolving the IPNS record

Callers are free to publish revisions as they please. Using [w3name](https://github.com/storacha/w3name) will enable fast resolution as well as IPFS DHT distribution.

Alternatively the Lit Action that mutates the record can publish it to w3name.
