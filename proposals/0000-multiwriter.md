
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
- merge conflict resolution (using the provided API), potentially using
  application-layer semantics

Before we go any further, a few definitions:

*Feed*: A hypercore feed: an append-only log of *Entries*, which can be
arbitrary data blobs.

*Database*: in this context, a Hyperdb key/value database. Built from several
Feeds (two Feeds per Writer).

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
replicated data type" ([CRDT][crdt]), and specifically an "Operation-based" (as
opposed to "State-based") CRDT.

[vc]: https://en.wikipedia.org/wiki/Vector_clock
[crdt]: https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type


## Core API
[api]: #api

A "node" is a data structure representing a database entry, including the
`key`, `value`, and feed that the entry is commited to.

`db.get(key)` (as described in the [hyperdb DEP][dep-hyperdb])
returns an array of nodes. If it is unambiguous what the consistent state of a
key is, the array will have only that one value. If there is a conflict
(ambiguity), multiple nodes will be returned. If a key has unambiguously been
removed from the database, a "null" or empty datatype is returned. If one
branch of a conflict has a deletion (but at least one of the others does not),
a node with the `deleted` flag will be returned; note that such "tombstone"
nodes can still have a `value` field, which may contain application-specific
metadata (such as a self-reported timestamp), which may help resolve the
conflict.

If multiple nodes are returned from a `get`, the writer should attempt to merge
the values (or chose one over the other) and write the result to the database
with `db.put(key, value)`. Client libraries can make this process more
ergonomic by accepting a helper function to select between multiple nodes.
Libraries can also offer an option to either directly return the value of a
single node (instead of the node itself), or raise an error; this is likely to
be more ergonomic for applications which do not intend to support multiple
writers per database.

`db.authorize(key)` will write metadata to the local feed authorizing the new
feed (corresponding to `key`) to be included in the database. Once authorized,
a feed may further authorize additional feeds (recursively).

`db.authorized(key)` (returning a boolean) indicates whether the given `key` is
an authorized writer to the hyperdb database.

At the time of this DEP there is no mechanism for revoking authorization.

[dep-hyperdb]: https://github.com/datprotocol/DEPs/blob/master/proposals/0004-hyperdb.md


## Scaling
[scaling]: #scaling

There is some overhead associated with each "writer" added to the feed,
impacting the number of files on disk, memory use, and the computational cost
of some lookup oprations. The design should easily accomodate dozens of
writers, and should scale to 1,000 writers without too much additional
overhead. Note that a large number of writers also implies a larger number and
rate of append operations, and additional network connections, which may cause
scaling issues on their own. More real-world experience and benchmarking is
needed in this area.


# Implementation Details
[reference-documentation]: #reference-documentation

The complete protobuf schemas for the hyperdb "Entry" and "InflatedEntry"
message types (as specified in the hyperdb DEP) are:


```
message Entry {
  required string key = 1;
  optional bytes value = 2;
  optional bool deleted = 3;
  required bytes trie = 4;
  repeated uint64 clock = 5;
  optional uint64 inflate = 6;
}

message InflatedEntry {
  message Feed {
    required bytes key = 1;
  }

  required string key = 1;
  optional bytes value = 2;
  optional bool deleted = 3;
  required bytes trie = 4;
  repeated uint64 clock = 5;
  optional uint64 inflate = 6;
  repeated Feed feeds = 7;
  optional bytes contentFeed = 8;
}
```

The fields of interest for multi-writer are:

- `clock`: a "vector clock" to record observed state at the time of writing the
  entry. Included in every Entry and InflatedEntry.
- `inflate`: a back-pointer to the entry index of the most recent InflatedEntry
  (containing a feed metadata change). Included in every Entry and
  InflatedEntry. Should not be included for the very first entry in a feed
  (which is an InflatedEntry).
- `feeds`: list of public keys for each writer's feed. Only included in
  InflatedEntry, and only when feeds have changed. Does include a
  self-reference to the current (local) Feed's key, always as the first
  element.

When serialized on disk in a SLEEP directory, the original feed is written
under `source/`. If the "local" feed is different from the original feed (aka,
local is not the "Original Writer"), it is written to `local/`. All other feeds
are written under directories prefixed `peers/<feed-discovery-key>/`.

## Feeds and Vector Clocks

At any point in time, each writer has a potentially unique view of the
"current" state of the database as a whole; this is the nature of real-world
distributed systems. For example, a given write might have the most recent
appends from one peer (eg, only seconds old), but be missing much older appends
from another (eg, days or weeks out of date). By having each writer include
metadata about their percieved state of the system as a whole in operations to
their Feed, all writers are able to collectively converge on an "eventually
consistent" picture of the database as whole (this process will be described in
the next section).

A writer's "current known state" of the database consists of the set of active
Feeds, and for each the most recent entry sequence number ("version"). This
state can be serialized as an array of integers, refered to as a [vector
clock](https://en.wikipedia.org/wiki/Vector_clock).

Each `put()` operation on the database appends a node to the writer's `local`
feed, and contains the writer's vector clock as observed at that time.
`InflatedEntry` nodes also contain a list of all known authorized Feeds;
inflated nodes only need to be written when the Feed list changes. Every
non-inflated entry contains a pointer back to the most recent inflated entry;
inflated entries themselves contain a pointer back to the previous inflated
entry (the first inflated entry has a null pointer). Elements of a vector clock
are ordered by the Feed list from the corresponding Inflated entry.

By convention, the order of Feed lists is to start with the writer's local
feed first, then proceed by the order in which Feeds were discovered. Note that
this ordering is not consistent across writers, only within the same feed.

As an example, if a node (non-inflated entry) had a vector clock of `[0, 2,
5]`, that would mean:

- when this node was written, the largest seq # in the writer's local fed was 0
- when this node was written, the largest seq # in the second known feed was 2
- when this node was written, the largest seq # in the third known feed was 5


## Multi-Feed Aware hyperdb
[multi-aware]: #multi-aware

The [hyperdb DEP](hyperdb-dep) specifies the procedures for lookup (`get()`)
and append (`put()`) operations to the database, as well as binary encoding
schemes for entry messages.

Note that the trie encoding specifies pointers in a `(feed, entry)` pair
format. The `feed` integer is an index into the most recent Feed list (found in
the most recent inflated entry; see the last section). When working with a
multi-writer hyperdb database, simply look up entries in the appropriate feed,
instead of only looking in the current feed. The next section ("Consistent
History") describes which entry (or entries) to start with instead of simply
assuming the most recent entry from the local feed.


## Consistent History
[consist-history]: #consist-history

The set of all appended nodes in all feeds of a hyperdb, and all the vector
clock pointers between them, forms a "directed acyclic graph" (DAG). Any node
which does not have another node pointing to it is called a "head" (this
terminology is similar to that used in git). At any point in time, an observed
copy of a database has one or more heads, each representing the top of a
tree-like graph. In the trivial case of a non-multi-writer hyperdb, there is
always only a single head: the most recent entry in the local feed. Just after
appending to the local feed, there is also always a single head, because that
node's vector clock will reference all know most recent entries from other
feeds. It is only when nodes are appended by separate writers who did not know
of the others' concurrent action (and then these changes are replicated) that
there are multiple heads.

When operating on the database (eg, executing a `get()` operation), all heads
must be considered. The lookup proceedure documented in the [hyperdb
DEP](hyperdb-dep) must be repeated for each head, and nodes returned
representing the set of all unique values.

The situation where a `get()` operation multiple heads returns different values
for the same key is called a "conflict" and requires a "merge" to resolve. Some
writer (either a human being or application-layer code) must decide on the
correct value for the key and write that value as a new entry (with a vector
clock that includes the previous heads). The procedure for chosing the best
value to use in a given conflict is sometimes easy to determine, but is
impossible to determine algorithmically in the general case. See the "Usage"
section for more details.


# Examples
[examples]: #examples

Let's say Alice starts a new hyperdb and writes two key/value entries to it:

```
// Operations
Alice: db.put('/foo/bar', 'baz')
Alice: db.put('/foo/2',   '{"json":3}')

// Alice's Feed
0 (key='/foo/bar', value='baz',
   vector_clock=[0], inflated=null, feeds=['a11ce...']) (InflatedEntry)
1 (key='/foo/2', value='{"json":3}',
   vector_clock=[0], inflated=0)

// Graph
Alice:  0  <---  1
```

The vector clock at `seq=1` points back to `seq=0`.

Next Alice *authorizes* Bob to write to the database. Internally, this means Alice
writes an Inflated entry to her feed that contains Bob's Feed (identified by his
public key) in her feed list.

```
// Operations
Alice: db.authorize('b0b123...')

// Alice's Feed
0 (key='/foo/bar', value='baz',
   vector_clock=[0], inflated=null, feeds=['a11ce...']) (InflatedEntry)
1 (key='/foo/2', value='{"json":3}',
   vector_clock=[0], inflated=0)
2 (key=null, value=null,
   vector_clock=[1], inflated=0, feeds=['a11ce...', 'b0b123...']) (InflatedEntry)

// Graph
Alice: 0  <---  1  <---  2
```

Bob writes a value to his feed, and then Alice and Bob sync. The result is:

```
// Operations
Bob: db.put('/a/b', '12)

// Alice's Feed
0 (key='/foo/bar', value='baz',
   vector_clock=[0], inflated=null, feeds=['a11ce...']) (InflatedEntry)
1 (key='/foo/2', value='{"json":3}',
   vector_clock=[0], inflated=0)
2 (key=null, value=null,
   vector_clock=[1], inflated=0, feeds=['a11ce...', 'b0b123...']) (InflatedEntry)

// Bob's Feed
0 (key='/a/b', value='12',
   vector_clock=[0], inflated=null, feeds=['b0b123...']) (InflatedEntry))

// Graph
Alice: 0  <---  1  <---  2
Bob  : 0
```

Notice that none of Alice's entries refer to Bob's, and vice versa. Neither has
written any entries to their feeds since the two became aware of each other.
Right now there are two "heads" of the graph: Alice's feed at seq 2, and Bob's
feed at seq 0. Any `get()` operations would need to descend from both heads,
though in this situation there would be no conflicts as the keys in the two
feeds are disjoint.

Next, Alice writes a new value, and her latest entry will refer to Bob's:

```
// Operations
Alice: db.put('/foo/hup', 'beep')

// Alice's Feed
0 (key='/foo/bar', value='baz',
   vector_clock=[0], inflated=null, feeds=['a11ce...']) (InflatedEntry)
1 (key='/foo/2', value='{"json":3}',
   vector_clock=[0], inflated=0)
2 (key=null, value=null,
   vector_clock=[1, null], inflated=0, feeds=['a11ce...', 'b0b123...']) (InflatedEntry)
3 (key='/foo/hup', value='beep',
   vector_clock=[2,0], inflated=2)

// Bob's Feed
0 (key='/a/b', value='12',
   vector_clock=[0], inflated=null, feeds=['b0b123...']) (InflatedEntry))


// Graph
Alice: 0  <---  1  <---  2  <--/  3
Bob  : 0  <-------------------/
```

Alice's latest feed entry now points to Bob's latest feed entry, and there is
only one "head" in the database. This means that any `get()` operations only
need to run once, starting at `seq=3` in Alice's feed.


# Security and Privacy Concerns
[privacy]: #privacy

As noted above, there is no existing mechanism for removing authorization for a
feed once added, and an authorized feed may recursively authorize additional
feeds. There is also no mechanism to restrict the scope of an authorized feed's
actions (eg, limit to only a specific path prefix). This leaves application
designers and users with few tools to control trust or access ("all or
nothing"). Care must be taken in particular if self-mutating software is being
distributed via hyperdb, or when action may be taken automatically based on the
most recent content of a database (eg, bots or even third-party tools may
publish publicly, or even take real-world action like controlling an electrical
relay).

There is no mechanism to remove malicious history (or any history for that
matter); if an authorized (but hostile) writer appends a huge number of key
operations (bloating hyperdb metadata size), or posts offensive or illegal
content to a database, there is no way to permanently remove the data without
creating an new database.

The read semantics of hyperdb are unchanged from hypercore: an actor does not
need to be "authorized" (for writing) to read the full history of a database,
they only need the public key.

As noted in other DEPs, a malicious writer can potentially execute a denial of
service (DoS) attack by appending hyperdb entries that for a cyclic loop of
references.


# Drawbacks
[drawbacks]: #drawbacks

Mutli-writer capability incurs a non-trivial increase in library, application,
and user experience complexity. For many applications, collaboration is an
essential feature, and the complexity is easily justified. To minimize
complexity for applications which do not need multi-writer features,
implementation authors should consider configuration modes which hide the
complexity of unused features. For example, by having an option to returning a
single node for a `get()` (and throw an error if there is a conflict), or a
flag to throw an error if a database unexpectedly contains more than a single
feed.

Two tasks (conflict merges and secure key distribution) are left to application
developers. Both of these are Hard Problems. The current design mitigates the
former by reducing the number of merge conflicts that need to be handled by an
application (aka, only the non-trivial ones need to be handled), and
implementation authors are encouraged to provide an ergonomic API for writing
conflict resolvers. The second problem (secure key distribution) is out of
scope for this DEP. It is hoped that at least one pattern or library will
emerge from the Dat ecosystem such that each application author doesn't need to
invent a solution from scratch.


# Rationale and alternatives
[alternatives]: #alternatives

Design goals for hyperdb (including the multi-writer feature) included:

- ability to execute operations (get, put) with a sparse (partial) replication
  of the database, using as few additional network requests as possible
- minimal on-disk and on-wire overhead
- implemented on top of an append-only log (to build on top of hypercore)

If a solution for core use cases like collaboration and multi-device
synchronization is not provided at a low level (as this DEP provides), each
application will need to invent a solution at a higher level, incuring
duplicated effort and a higher risk of bugs.

As an alternative to CRDTs, Operational Transformation (OT) has a reputation
for being more difficult to understand and implement.


# Unresolved questions
[unresolved]: #unresolved-questions

What is the actual on-disk layout (folder structure), if not what is documented
here?

The local feed's sequence number could skipped from vector clocks, because it's
implied by the sequence number of the hypercore entry itself. Same with the key
in the feed list (for inflated entries). In both cases, the redundant data is
retained for simplicity.

If there are multiple heads, but they all return the same `value` for a `get()`
operation, how is it decided which `node` will be returned? AKA, values are the
same, but node metadata might not be (order of vector clock, etc).

Suspect that some details are off in the example: shouldn't the InflatedEntry
authorizing a new feed include a vector clock reference to a specific seq in
that feed? Should new local (not yet authorized) feeds reference
their source in an initial InflatedEntry (eg, Bob at seq=0)? Should the first
InflatedEntry in a feed point to itself in it's vector clock?

# Changelog
[changelog]: #changelog

As of March 2018, Mathias Buus (@mafintosh) is leading development of a hyperdb
nodejs module on [github](https://github.com/mafintosh/hyperdb), which includes
multi-writer features and is the basis for this DEP.

Jim Pick (@jimpick) has been an active contributor working out multi-writer details.

- 2017-12-06: @noffle publishes `ARCHITECTURE.md` overview in the
  [hyperdb github repo][arch_md]
- 2018-06-XX: First partial draft submitted for review

[arch_md]: https://github.com/mafintosh/hyperdb/blob/master/ARCHITECTURE.md
