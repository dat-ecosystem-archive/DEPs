
Title: **DEP-0000: Multi-Writer**

Short Name: `0000-multiwriter`

Type: Standard

Status: Undefined (as of 2018-03-XX)

Github PR: (add HTTPS link here after PR is opened)

Authors:
[Bryan Newbold](https://github.com/bnewbold),
[Stephen Whitmore](https://github.com/noffle),
[Mathias Buus](https://github.com/mafintosh)


# Summary
[summary]: #summary

Multi-Writer is a set of schema, API, and feature extentions to multiple agents
(users, devices, or software) to write to the same HyperDB feed. By building on
top of this abstraction layer, future versions of hyperdrive and Dat will gain
these features.

Mechanisms for distributed consistency and granting trust are specified here;
the need for merge conflict algorithms and secure key distribution are
mentioned but specific solutions are not specified.


# Motivation
[motivation]: #motivation

The current hypercore/Dat ecosystem currently lacks solutions two fundamental
use cases:

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


# Semantics and Usage
[usage-documentation]: #usage-documentation

TODO: semantics, terminology

TODO: things left to application developers (secure key distribution, non-trivial merge resolution)

TODO: brief note on scaling properties (eg, reasonable numbers of feeds per database)


# Implementation
[reference-documentation]: #reference-documentation

The protobuf schema fields of interest for multi-writer (from the "Node"
message type specified in the HyperDB DEP) are:

- `seq`: the sequence number of this entry in the owner's hypercore. 0 is the
  first, 1 the second, and so forth.
- `feed`: the ID of the hypercore writer that wrote this
- `clock`: vector clock to determine node insertion causality

## Directed acyclic graph

The combination of all operations performed on a HyperDB by all of its members
forms a DAG (*directed acyclic graph*). Each write to the database (setting a
key to a value) includes information to point backward at all of the known
"heads" in the graph.

To illustrate what this means, let's say Alice starts a new HyperDB and writes 2
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

Now Alice *authorizes* Bob to write to the HyperDB. Internally, this means Alice
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

TODO: Why should we *not* do this?


# Rationale and alternatives
[alternatives]: #alternatives

TODO:

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?


# Unresolved questions
[unresolved]: #unresolved-questions

TODO:

- What parts of the design do you expect to resolve through the DEP consensus process before this gets merged?
- What parts of the design do you expect to resolve through implementation and code review, or are left to independent library or application developers?
- What related issues do you consider out of scope for this DEP that could be addressed in the future independently of the solution that comes out of this DEP?


# Changelog
[changelog]: #changelog

As of March 2018, Mathias Buus (@mafintosh) is leading development of a hyperdb
nodejs module on [github](https://github.com/mafintosh/hyperdb), which includes
multi-writer features and is the basis for this DEP.

- 2017-12-06: @noffle publishes `ARCHITECTURE.md` overview in the
  [hyperdb github repo][arch_md]
- 2018-03-XX: First partial draft submitted for review

[arch_md]: https://github.com/mafintosh/hyperdb/blob/master/ARCHITECTURE.md
