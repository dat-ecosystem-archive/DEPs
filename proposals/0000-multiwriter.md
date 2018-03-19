
Title: **DEP-0000: Multi-Writer**

Short Name: `0000-multiwriter`

Type: Standard

Status: Undefined (as of 2018-03-XX)

Github PR: [Draft](https://github.com/datprotocol/DEPs/pull/10)

Authors:
[Bryan Newbold](https://github.com/bnewbold),
[Stephen Whitmore](https://github.com/noffle),
[Mathias Buus](https://github.com/mafintosh)


# Summary
[summary]: #summary

Multi-Writer is a set of schema, API, and feature extentions to allow multiple
agents (users, devices, or software) to write to the same hyperdb database. By
building on top of this abstraction layer, future versions of hyperdrive and
Dat will gain these features.

Mechanisms for distributed consistency and granting trust are specified here;
the need for merge conflict algorithms and secure key distribution are
mentioned but specific solutions are not specified.

This DEP forms the second half of the hyperdb specification; the first half
covered only the key/value database aspects of hyperdb.


# Motivation
[motivation]: #motivation

The current hypercore/Dat ecosystem currently lacks solutions for two
fundamental use cases:

- individual users should be able to modify distributed archives under their
  control from multiple devices, at a minimum to prevent loss of control of
  content if a single device (containing secret keys) is lost
- contributions from and collaboration between multiple users on a single
  archive or database should be possible, with appropriate trust and access
  control semantics

Access to a single secret key is currently required to make any change to a
hypercore feed, and it is broadly considered best practice not to distribute
secret keys between multiple users or multiple devices. In fact, the current
hypercore implementation has no mechanism to resolve disputes or recover if
multiple agents used the same secret key to append to the same feed.

Solutions to these two use cases are seen as essential for many current and
future Dat ecosystem applications.


# Concepts, Behavior, and Usage
[usage-documentation]: #usage-documentation

The multi-writer features of hyperdb are implemented by storing and replicating
the contributions of each writer in a separate hypercore feed. This
specification concerns itself with the details of how changes from multiple
feeds (which may be written and replicated concurrently or asynchronously) are
securely combined to present a unified key/value interface.

The following related concerns are explicitly left to application developers to
design and implement:

- secure key distribution and authentication (eg, if a friend should be given
  write access to a hyperdb database, how is that friend's feed key found and
  verified?)
- merge conflict resolution, potentially using application-layer semantics

Before we go any further, a few definitions:

*Feed*: A hypercore feed: an append-only log of *Entries*, which can be
arbitrary data blobs. Hyperdb is built on top of several Feeds.

*Writer*: a user (or user controlled device or software agent) that has a
distinct feed with a public/private key pair, and thus the ability to append
hypercore entries and "write" changes to their version of the database.

*Original Writer*: the writer who created a given hyperdb database in the form
of the *Original Feed*. The public key of the original feed is the one used to
reference the database as a collection of feeds (eg, for the purpose of
discovery).

At a high level, multi-writer hyperdb works by having existing authorized
writers (starting with the original writer) include authorization of new
writers by appending metadata to their own feed which points to the new feeds
(by public key). Each entry in each writer's feed contains "clock" metadata
that records the known state of the entire database (all writers) seen from the
perspective of that writer at the time they created the entry, in the form of
"clock" version pointers. This metadata (a "[vector clock][vc]") can be used by
other writers to resolve (or at least identify) conflicting content in the
database. The technical term for this type of system is a "Conflict-free
replicated data type" ([CRDT][crdt]).

[vc]: https://en.wikipedia.org/wiki/Vector_clock
[crdt]: https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type


## API
[api]: #api

The `db.get(key)` method described in the [hyperdb DEP][dep-hyperdb] actually
returns an array of values. If it is unambiguous what the consistent value is,
the array will have only that one value. If there is a conflict, multiple
values will be returned. If multiple values are returned, the writer should
attempt to merge the values (or chose one over the other) and write the result
to the database with `db.put(key, value)`.

`db.authorize(key)` will write metadata to the local feed authorizing the new
feed (corresponding to `key`) to be included in the database.

<!-- TODO: update link -->
[dep-hyperdb]: https://github.com/bnewbold/dat-deps/blob/dep-hyperdb/proposals/0000-hyperdb.md


## Scaling
[scaling]: #scaling

TODO: brief note on scaling properties (eg, reasonable numbers of writers per
database)


# Implementation
[reference-documentation]: #reference-documentation

The complete protobuf schemas for the hyperdb "Entry" and "InflatedEntry"
message types (as specified in the hyperdb DEP) are:


```
message Entry {
  required string key = 1;
  optional bytes value = 2;
  required bytes trie = 3;
  repeated uint64 clock = 4;
  optional uint64 inflate = 5;
}

message InflatedEntry {
  message Feed {
    required bytes key = 1;
  }

  required string key = 1;
  optional bytes value = 2;
  required bytes trie = 3;
  repeated uint64 clock = 4;
  optional uint64 inflate = 5;
  repeated Feed feeds = 6;
  optional bytes contentFeed = 7;
}
```

The fields of interest for multi-writer are:

- `clock`: a "vector clock" to record observed state at the time of writing the
  entry. Included in every Entry and InflatedEntry.
- `inflate`: a back-pointer to the entry index of the most recent InflatedEntry
  (containing a feed metadata change). Included in every Entry and
  InflatedEntry.
- `feeds`: list of public keys for each writer's feed. Only included in
  InflatedEntry, and only when feeds have changed.

When serialized on disk in a SLEEP directory, the original feed is written
under `./source/`. If the "local" feed is different from the original feed, it
is written to `./local/`. All other feeds are written under directories
prefixed `./peers/<feed-discovery-key>/`.

TODO: the above disk format is incorrect?

## Directed acyclic graph
[dag]: #dag

The combination of all operations performed on a hyperdb by all of its members
forms a DAG (*directed acyclic graph*). Each write to the database (setting a
key to a value) includes information to point backward at all of the known
"heads" in the graph.

To illustrate what this means, let's say Alice starts a new hyperdb and writes 2
values to it:

```
// Feed

0 (/foo/bar = 'baz')
1 (/foo/2   = '{ "some": "json" }')


// Graph

Alice:  0  <---  1
```

Where sequence number 1 (the second entry) refers to sequence number 0 on the
same feed (Alice's).

Now Alice *authorizes* Bob to write to the hyperdb. Internally, this means Alice
writes a special message to her feed saying that Bob's feed (identified by his
public key) should be read and replicated in by other participants. Her feed
becomes

```
// Feed

0 (/foo/bar = 'baz')
1 (/foo/2   = '{ "some": "json" }')
2 (''       = '')


// Graph

Alice: 0  <---  1  <---  2
```

Authorization is formatted internally in a special way so that it isn't
interpreted as a key/value pair.

Now Bob writes a value to his feed, and then Alice and Bob sync. The result is:

```
// Feed

//// Alice
0 (/foo/bar = 'baz')
1 (/foo/2   = '{ "some": "json" }')
2 (''       = '')

//// Bob
0 (/a/b     = '12')


// Graph

Alice: 0  <---  1  <---  2
Bob  : 0
```

Notice that none of Alice's entries refer to Bob's, and vice versa. This is
because neither has written any entries to their feeds since the two became
aware of each other (authorized & replicated each other's feeds).

Right now there are two "heads" of the graph: Alice's feed at seq 2, and Bob's
feed at seq 0.

Next, Alice writes a new value, and her latest entry will refer to Bob's:

```
// Feed

//// Alice
0 (/foo/bar = 'baz')
1 (/foo/2   = '{ "some": "json" }')
2 (''       = '')
3 (/foo/hup = 'beep')

//// Bob
0 (/a/b     = '12')


// Graph

Alice: 0  <---  1  <---  2  <--/  3
Bob  : 0  <-------------------/
```

Because Alice's latest feed entry refers to Bob's latest feed entry, there is
now only one "head" in the database. That means there is enough information in
Alice's seq=3 entry to find any other key in the database. In the last example,
there were two heads (Alice's seq=2 and Bob's seq=0); both of which would need
to be read internally in order to locate any key in the database.

Now there is only one "head": Alice's feed at seq 3.


## Authorization

The set of hypercores are *authorized* in that the original author of the first
hypercore in a hyperdb must explicitly denote in their append-only log that the
public key of a new hypercore is permitted to edit the database. Any authorized
member may authorize more members. There is no revocation or other author
management elements currently.


## Vector clock

Each node stores a [vector clock](https://en.wikipedia.org/wiki/Vector_clock) of
the last known sequence number from each feed it knows about. This is what forms
the DAG structure.

A vector clock on a node of, say, `[0, 2, 5]` means:

- when this node was written, the largest seq # in my local fed is 0
- when this node was written, the largest seq # in the second feed I have is 2
- when this node was written, the largest seq # in the third feed I have is 5

For example, Bob's vector clock for Alice's seq=3 entry above would be `[0, 3]`
since he knows of her latest entry (seq=3) and his own (seq=0).

The vector clock is used for correctly traversing history. This is necessary for
the `db#heads` API as well as `db#createHistoryStream`.


# Examples
[examples]: #examples

TODO:


# Security and Privacy Concerns
[privacy]: #privacy

TODO:


# Drawbacks
[drawbacks]: #drawbacks

Significant increase in implementation, application, and user experience
complexity.

Two hard problems (merges and secure key distribution) are left to application
developers.


# Rationale and alternatives
[alternatives]: #alternatives

TODO:

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?


# Unresolved questions
[unresolved]: #unresolved-questions

What is the technical term for the specific CRDT we are using?
"Operation-based"?

If there are conflicts resulting in ambiguity of whether a key has been deleted
or has a new value, does `db.get(key)` return an array of `[new_value, None]`?

What is a reasonable large number of writers to have in a single database?
Write "Scaling" section.


# Changelog
[changelog]: #changelog

As of March 2018, Mathias Buus (@mafintosh) is leading development of a hyperdb
nodejs module on [github](https://github.com/mafintosh/hyperdb), which includes
multi-writer features and is the basis for this DEP.

- 2017-12-06: @noffle publishes `ARCHITECTURE.md` overview in the
  [hyperdb github repo][arch_md]
- 2018-03-XX: First partial draft submitted for review

[arch_md]: https://github.com/mafintosh/hyperdb/blob/master/ARCHITECTURE.md
