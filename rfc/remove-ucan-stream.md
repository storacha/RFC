# RFC: Remove the UCAN stream

## Authors

- [Alan Shaw](https://github.com/alanshaw), [Storacha Network](https://storacha.network/)

## Introduction

All invocations and receipts are put on the UCAN stream - currently an AWS Kinesis instance.

The original idea of the UCAN stream was to act as an ordered log of events where (potentially) all tasks could be distributed and performed async by a decentralized network. It would allow invocations to be accepted very quickly, and tasks of any size to be completed without time limitations. Additionally the stream acted as a backup of events that could be replayed at any time to restore the current state of the system. Sadly(?) this original vision has not been realized. We instead perform most actions synchronously when they are invoked and use the stream for just a handful of tasks that relate only to post-processing.

The UCAN stream exists for 2 reasons currently:

1. Billing for space usage - we inspect receipts for certain invocations and add usage delta records per space in order to allow space size to be accounted for and billed.
2. Metrics - we currently store _counts_ for certain invocations and for invocations that store or removed data, the accumulated _size_ of data added or removed.

We've observed a few issues with this:

* Stream processing errors lead to incorrect counts that are never rectified.
* In the specific case of billing, we've discovered that some receipts (that are required for billing) are added onto the stream multiple times, due to the fact that they are embedded in multiple responses. Specifically the `web3.storage/blob/allocate` receipt is added onto the stream when a blob is allocated, and then again when it is included in the response for `space/blob/add`. Since [this change](https://github.com/storacha/w3infra/pull/380), we have been double counting space usage additions.
* Async post-processing tasks i.e. tasks that don't have a direct effect on the outcome of the original invocation, can be easily lost since there is no record of them being scheduled, succeeding or failing.
* Async tasks that depend on the receipt of a different invocation are brittle to changes in the receipt. The tasks are often "far away" from changes to the receipt they depend on and changes to the receipt may not be known to break the task until much later (or worst case at runtime in production).
* Unless the async tasks account stricly for receipts they have seen, they are brittle to duplication.
* Tasks that depend on the receipt of a different invocation can exert influence over it's content. For example, in billing we discovered that storing the same blob multiple times was causing double counts. This is undesirable - a user should not be charged multiple times for the same item of data. At the time the _easiest_ solution was to alter the receipt the billing service relied upon to include the number of allocated bytes. Essentially if the blob already exists in the space then the allocated bytes are 0, but if the blob does not then the allocated bytes are equal to the size of the blob. This represents a loose coupling between `billing` and `upload-api` sub-systems which is brittle for reasons stated above. A better solution might be to define specific usage invocations for billing and to invoke them at the appropriate point when and additon is known to affect space size.

## Proposal

In general the proposal is to transition these tasks from async to sync. This gives us better visibility over what tasks need to be performed and when, due to the fact that they are executed at the point they are required, not later. It makes explicit the coupling between sub-systems where it was previously loose and implicit. It also surfaces failures earlier and provided we ensure atomicity, will maintain system consistency.

We SHOULD be careful to ensure atomic writes when an invocation writes to multiple places. In dynamodb this can easily be achieved using `TransactWriteItems`, or alternatively we can "rollback" succeeded tasks when others fail.

The risk is that our invocations take a little longer to execute, but consistency within the system is more desirable than speed.

The work in [Consolidating `w3up` + `upload-service`](https://github.com/storacha/RFC/blob/main/rfc/consolidating-w3up-and-upload-service.md) actually includes porting a change made to `upload-service-infra` that switches metrics recording to be done at the time events happen, rather than asynchronously later. This was done as an experimental effort towards the proposal in this RFC. Note: this does not include any change to metrics collection for Store Protocol invocations.

### Short-term

Switch the UCAN stream handler to use `web3.storage/blob/accept` receipts, which do not currently appear in another invocation response and thus are not duplicated on the stream.

This re-exposes the bug where two uploads of the same item are double counted, but it is categorically better than double counting _every_ upload.

### Mid-term

1. Insert space usage deltas directly to DynamoDB table from the `BlobRegistry` at the point where items are added or removed.
2. Remove billing service UCAN stream handler.
3. Post Store Protocol decommissioning, remove UCAN stream entirely from the code base.

### Long-term

Define a space usage invocation (e.g. `space/usage/record`) and actually invoke it when we receive a `blob/accept` receipt. In the same way we have a [defined invocation for egress events](https://github.com/storacha/upload-service/blob/7fe172465c6692644815a330f677879c4fb616e8/packages/capabilities/src/space.js#L78-L94).
