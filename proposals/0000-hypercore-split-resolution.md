
Title: **DEP-0000: Hypercore Split Resolution**

Short Name: `0000-hypercore-split-resolution`

Type: Standard

Status: Draft (as of 2018-10-02)

Github PR: (add HTTPS link here after PR is opened)

Authors: Paul Frazee


# Summary
[summary]: #summary

The Hypercore data-structure is an append-only log which depends on maintaining a linear and unbranching history in order to encode the state-changes of its data content. A "split" event occurs when the log branches, losing its linear history. This spec provides a process for resolving "splits" in a Hypercore log.


# Motivation
[motivation]: #motivation

Hypercore requires a linear history in order to function correctly. A "split" event is a fatal corruption: peers which encounter a split will stop replication, causing the Hypercore to lose utility.

In cases with a strict security requirement this might be useful, but the append-only invariant can be difficult to maintain for users who migrate their dats between devices or restore from backups. For most users, it would be more desirable to risk losing some writes than to risk losing an entire dat due to a split.

This DEP specifies a process for recovering from splits so that users can safely backup or transfer their dats.


# Usage Documentation
[usage-documentation]: #usage-documentation

Hypercore's APIs will provide a method for registering a "split handler." The split handler will be implemented by the application using the Hypercore. It may have a standard definition provided by a higher-level data structure such as HyperDB or Hyperdrive.

The "split handler" function will receive the `seq` number, `blockA`, and `blockB`. It will be expected to return `-1` to accept `blockA` or `1` to accept `blockB`. If `0` is returned, no resolution occurs and the new block is rejected (the current behavior). When a split is resolved, any subsequent messages after the rejected block will also be rejected.

It's expected that the split handler will read the contents of the blocks in order to make decisions about a split. In the case of Hyperdrive, for instance, a split might be resolved using the timestamp of the write.


# Reference Documentation
[reference-documentation]: #reference-documentation

A "split" is defined as a Hypercore which has two or more valid messages with the same sequence number.

"Split resolution" will occur during replication when a "split" is detected due to a received block. Split resolution should not occur during local writes. (Conflicting local writes should be rejected outright).

The split handler must be constructed in such a way that all peers will come to the same decisions independently. Peers should not make a decision based on information such as "time that the block was received," since it is not global knowledge. By only using global information, a split can be reliably resolved across the network.


# Drawbacks
[drawbacks]: #drawbacks

"Split resolution" will cause data loss as some part of the history must be discarded. If the managing software is not careful, this can result in massive data loss (e.g. if the split occurs at the first message during recovery). To limit this potential, the managing software can query the network for the latest history in cases where a split is likely (such as a backup recovery process).

"Split resolution" will cause the append-only invariant of Hypercore logs to be optional. This means that file history and versions will not be immutable.


# Rationale and alternatives
[alternatives]: #alternatives

- Previously we've discussed a "major version pointer" which made it possible possible to recover from a split by publishing a new history ([discussion](https://github.com/datprotocol/DEPs/issues/31)). The "major version pointer" is a less efficient solution as it requires a wholly new history to recover, had its own edge-cases which were difficult to recover from, and was rejected for being too complex.


# Unresolved questions
[unresolved]: #unresolved-questions


# Changelog
[changelog]: #changelog

- 2018-10-02: First complete draft submitted for review

