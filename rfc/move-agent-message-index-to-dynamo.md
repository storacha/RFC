# RFC: Move agent message index to dynamo

## Authors

- [Alan Shaw](https://github.com/alanshaw), [Storacha Network](https://storacha.network/)

## Introduction

Every request and response CAR is written to S3. Additionally for every request and response we write a "symlink" to S3, sometimes multiple. A "symlink" is a 0 byte file where the information is encoded in the key.

The symlinks map invocations and receipts to the request/response CARs that they can be found in. They are index information. Note that it's possible for a single request to contain multiple invocations or multiple receipts. In this case we make _multiple_ writes to S3 for a given request/resposne.

## Proposal

S3 write requests are our biggest expense. Lets move the index information to DyanmoDB. It allows batched writes (faster) and cost per write is cheaper $0.625 per million vs $5 per million.

Note: DyanmoDB storage costs are more expensive than S3, but the cost difference is likely negligable due to the small amount of data being stored.
