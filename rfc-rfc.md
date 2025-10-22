# RFC: Storacha RFCs

## 

## Authors

- [Travis Vachon](https://github.com/travis), [Storacha Network](https://storacha.network/)

## Introduction

The Storacha "Request For Comments" process is modeled after the ["Internet Engineering Task Force" (IETF) process of the same name](https://www.ietf.org/process/rfcs/) both as an homage to this legendary standards body and in recognition of the fact that our team aspires to create open source, standardized protocols for decentralized data storage and retrieval that have the potential to outlast our team and organization. Despite this aspiration, however, our team is both culturally and practically very different from the IETF and as such the customs and practices of the IETF's RFC process are not necessarily well suited to our needs. 

This document lays out guidelines for the creation of Storacha RFC documents. We describe the purpose of a Storacha RFC, discuss the various types of RFCs and provide templates and guidelines for RFC authors to both aid in their creation and help provide consistency between them. We also discuss both existing and desired processes for managing RFCs through their lifecycles and building consensus across our team toward the concrete actions RFCs propose.

## The Purpose of Storacha RFCs

Storacha RFCs serve a variety of purposes including:

1) Building consensus on courses of actions that could face blocking issues from other team members
2) To invite brainstorming of ideas that solve underlying issues in a more simpler or more elegant way
3) To ensure everyone on the team is aware of decisions and techniques so they can apply them in their own work
4) Laying groundwork for ideas that will eventually become formal specifications
5) Discussion of decisions that "ripple across our architecture"
6) Discussion of new patterns we expect to use in more than one service in our system

The bar for these purposes does not need to be high - if you suspect it may serve one or more of the purposes above, please feel free to create an RFC to drive discussion forward.

## Should I Write an RFC?

Some criteria to consider when deciding whether to write an RFC include:

1) Does this decision commit our work to a path that we may not be able to easily walk back in the future?
2) Am I proposing a protocol, where once something is in the world we will need to deal with backwards compatibility?
3) Is this an architectural decision that will impact multiple parts of our system?
4) Will this work need to be parallelized and therefore require multiple team members to work in close syncrony?
5) Is the proposed change too complex for a quick spike that could de-risk the decision?
6) Is there a large degree of ambiguity, where several different approaches might work?

If you answer "yes" to one or more, consider writing an RFC.

## RFC Status

> It is a regrettably well spread misconception that publication as an  RFC provides some level of recognition. It does not, or at least not any more than the publication in a regular journal. In fact, each RFC has a status, relative to its relation with the Internet standardization process: Informational, Experimental, or Standards Track (Proposed Standard, Draft Standard, Internet Standard), or Historic. This status is reproduced on the first page of the RFC itself, and is also documented in the periodic "Internet Official Protocols Standards" RFC (STD 1). But this status is sometimes omitted from quotes and references, which may feed the confusion.

https://datatracker.ietf.org/doc/html/rfc1796

Inspired by the IETF, we give RFCs a "status" that helps reviewers contextualize information in the RFC. RFCs can and should change status over time - an Informational RFC may become an Experimental RFC if it has moved into prototyping, and may become "Standards Track" if it becomes widely used and appropriate for external use. Statuses include: 

Informational
Experimental
Best Current Practice
Standards Track
Historical

"Standards Track" is broken down into 2 sub-statuses:

Proposed Standard
Published Standard

With heavy inspiration from https://www.ietf.org/process/rfcs/ the statuses are described below.

### Informational

Informational RFCs are published for the general information of the Storacha community, and do not represent consensus or recommendations for our team or community.

### Experimental 

Experimental RFCs are published as part of some research or development effort. Such a specification is published for the general information of the Storacha community and as an archival record of the work.

### Best Current Practice

Best Current Practice RFCs document Storacha processes as agreed to by our community and are intended to help our team and community members to follow some common guidelines for policies and operations.

### Proposed Standard

The first stage of our standards process. All Storacha protocols intended to serve as externally facing standards start here. Moving an RFC into "Proposed Standard" suggests we would like third parties to engage and potentially implement the RFC and represents a commitment on our part to sheparding this process.

### Published Standard

The second and final stage of our standards process. Moving an RFC to this stage represents a commitment on the part of our team to maintain backwards compatibility with the published standard indefinitely.

### Historical

The Historical status is for RFCs that are no longer current - this may mean we have dropped support for a standard or that the information in the RFC is either no longer used or no longer best practice.

## RFC Tooling

In order to ensure that RFCs are widely and consistently reviewed and don't get lost or stalled during review, we maintain a number of team and community processes and practices. These include:

### GitHub Pull Requests

RFCs should be submitted as Pull Requests to the https://github.com/storacha/rfc repository. The `storacha/developers` team should be added as a reviewer. Additionally, the RFC author should add individual developers from whom they would like especially close review as reviewers.

Discussion of an RFC should take place in the Pull Request using standard GitHub Pull Request tooling.

Developers SHOULD indicate their signoff on an RFC by "approving" the PR.

### Stakeholder Notifications

Developers SHOULD configure GitHub PR notifications so they receive updates about PRs they care about. Developers MAY mute discussions they are not interested in particupating in.

Developers MAY opt to receive notifications about Pull Requests in Discord (TODO how?). Additionally, we maintain a channel in Discord where notifications about activity in the https://github.com/storacha/rfc repository are posted. (TODO also how?)

