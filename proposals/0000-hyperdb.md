
Title: **DEP-0000: HyperDB**

Short Name: `0000-hyperdb`

Type: Standard

Status: Undefined (as of 2018-03-XX)

Github PR: (add HTTPS link here after PR is opened)

Authors: [Stephen Whitmore](https://github.com/noffle), [Bryan Newbold](https://github.com/bnewbold)


# Summary
[summary]: #summary

HyperDB is a new abstraction layer providing a general purpose distributed
key/value store over the Dat protocol. It is an iteration on the hyperdrive
directory tree implementation, building top of the hypercore append-only log
abstraction layer. Keys are path-like strings (eg, `/food/fruit/kiwi`), and
values are arbitrary binary blobs (generally under a megabyte).

Hyperdrive (used by the Dat application) is expected to be re-implemented on
top of HyperDB for improved performance with many files (eg, millions). The
hyperdrive API should be largely unchanged, but the `metadata` format will be
backwards-incompatible.


# Motivation
[motivation]: #motivation

HyperDB is expected to drastically improve performance of dat clients when
working with archives containing tens of thousands of files in single
directories. This is a real-world bottleneck for several current users, with
basic local actions such as adding a directory taking an unacceptably long time
to complete.

A secondary benefit is to refactor the [trie][trie]-structured key/value API
out of hyperdrive, allowing third party code to build applications directly on
this abstraction layer.

[trie]: https://en.wikipedia.org/wiki/Trie


# Usage Documentation
[usage-documentation]: #usage-documentation

*This section describes HyperDB's interface and behavior in the abstract for
application programmers. It is not intended to be exact documentation of any
particular implementation (including the reference Javascript module).*

HyperDB is structured to be used much like a traditional hierarchical
filesystem. A value can be written and read at locations like `/foo/bar/baz`,
and the API supports querying or tracking values at subpaths, like how watching
for changes on `/foo/bar` will report both changes to `/foo/bar/baz` and also
`/foo/bar/19`.

Lower-level details of the hypercore append-only log, disk serialization, and
networked synchronization features that HyperDB builds on top of are not
described in detail here; see the [DEP repository][deps]. Multi-writer
hypercore semantics are also not discussed in this DEP.

[deps]: https://github.com/datprotocol/DEPs

A HyperDB database instance can be represented by a single hypercore feed (or
several feeds in a multi-writer context), and is named, referenced, and
discovered using the public and discovery keys of the hypercore feed (or the
original feed if there are several). In a single-writer configuration, only a
single node (holding the secret key) can mutate the database (eg, via `put` or
`delete` actions).

**Keys** can be any UTF-8 string. Path segments are separated by the forward
slash character (`/`). Repeated slashes (`//`) are disallowed. Leading and
trailing `/` are optional in application code: `/hello` and `hello` are
equivalent. A key can be both a "path segment" and key at the same time; eg,
`/a/b/c` and `/a/b` can both be keys at the same time.

**Values** can be any binary blob, including empty (of zero length). For
example, values could be UTF-8 encoded strings, JSON encoded objects, protobuf
messages, or a raw `uint64` integer (of either endian-ness). Length is the only
form of type or metadata stored about the value; deserialization and validation
are left to library and application developers.

## Core API Semantics

A `db` is instantiated by opening an existing hypercore with hyperdb content
(read-only, or optionally read-write if the secret key is available), or
creating a new one. A handle could represent any specific revision in history,
or the "latest" revision.

`db.put(key, value)`: inserts `value` (arbitrary bytes) under the path `key`.
Requires read-write access.  Returns an error (eg, via callback) if there was a
problem.

`db.get(key)`: Reading a non-existant `key` is an error. Read-only.

`db.delete(key)`: Removes the key from the database. Deleting a non-existant
key is an error. Requires read-write access.

`db.list(prefix)`: returns a flat (not nested) list of all keys currently in
the database under the given prefix. Prefixes operate on a path-segment basis:
`/ab` is not a valid prefix for key `/abcd`, but is valid for `/ab/cd`. If the
prefix does not exist, returns an empty list. The order of returned keys is
implementation (or configuration) specific. Read-only.

If the hypercore underlying a hyperdb is only partially replicated, behavior is
implementation-specific. For example, a `get()` call could block until the
relevant value is replicated, or the implementation could return an error.

An example pseudo-code session working with a database might be:

    db.put('/life/animal/mammal/kitten', '{"cuteness": 500.3}')
    db.put('/life/plant/bush/bananna', '{"delicious": 103.4}')
    db.delete('/life/plant/bush/bananna')
    db.put('/life/plant/tree/bananna', '{"delicious": 103.4}')
    db.get('/life/animal/mammal/kitten')
    => {"cuteness": 500.3}
    db.list('/life/')
    => ['/life/animal/mammal/kitten', '/life/plant/tree/bananna']

# Reference Documentation
[reference-documentation]: #reference-documentation

A HyperDB hypercore feed typically consists of a sequence of entries, all of
which are protobuf-encoded `Node` messages. Higher-level protocols may make
exception to this: for example, hyperdrive reserves the first entry of the
(`metadata`) feed for a special entry that refers to the second (`content`)
feed.

The `Node` protobuf message schema is:

```
  message Node {
    optional string key = 1;
    optional bytes value = 2;
    repeated uint64 clock = 3;
    optional bytes trie = 4;
    optional uint64 feedSeq = 6;
  }
```

TODO(mafintosh): where is this schema actually defined for the `next` branch of
the hyperdb repo?

`key` and `value` are as expected. A non-existant `value` indicates that this
message is a deletion (see below). The `clock` element is reserved for use in
the forthcoming `multi-writer` standard, not described here. An empty list is
the safe (and expected) value for `clock` in single-writer use cases.

## Incremental Index

HyperDB builds an *incremental index* with every new key/value pairs ("nodes")
written. This means a separate data structure doesn't need to be maintained
elsewhere for fast writes and lookups: each node written has enough information
to look up any other key quickly and otherwise navigate the database.

Each node stores the following basic information:

- `key`: the key that is being created or modified. e.g. `/home/sww/dev.md`
- `value`: the value stored at that key.
- `feedSeq`: the sequence number of this entry in the hypercore. 0 is the first, 1
  the second, and so forth.
- `path`: a 2-bit hash sequence of the key's components
- `trie`: a navigation structure used with `path` to find a desired key

### Prefix trie

Given a HyperDB with hundreds of entries, how can a key like `/a/b/c` be looked
up quickly?

Each node stores a *prefix [trie](https://en.wikipedia.org/wiki/Trie)* that
assists with finding the shortest path to the desired key.

When a node is written, its *prefix hash* is computed. This done by first
splitting the key into its components (`a`, `b`, and `c` for `/a/b/c`), and then
hashing each component into a 32-character hash, where one character is a 2-bit
value (0, 1, 2, or 3). The `prefix` hash for `/a/b/c` is

```js
node.path = [
1, 2, 0, 1, 2, 0, 2, 2, 3, 0, 1, 2, 1, 3, 0, 3, 0, 0, 2, 1, 0, 2, 0, 0, 2, 0, 0, 3, 2, 1, 1, 2,
0, 1, 2, 3, 2, 2, 2, 0, 3, 1, 1, 3, 0, 3, 1, 3, 0, 1, 0, 1, 3, 2, 0, 2, 2, 3, 2, 2, 3, 3, 2, 3,
0, 1, 1, 0, 1, 2, 3, 2, 2, 2, 0, 0, 3, 1, 2, 1, 3, 3, 3, 3, 3, 3, 0, 3, 3, 2, 3, 2, 3, 0, 1, 0,
4 ]
```

Each component is divided by a newline. `4` is a special value indicating the
end of the prefix.

#### Example

Consider a fresh HyperDB. We write `/a/b = 24` and get back this node:

```js
{ key: '/a/b',
  value: '24',
  trie: [],
  seq: 0,
  path:
   [ 1, 2, 0, 1, 2, 0, 2, 2, 3, 0, 1, 2, 1, 3, 0, 3, 0, 0, 2, 1, 0, 2, 0, 0, 2, 0, 0, 3, 2, 1, 1, 2,
     0, 1, 2, 3, 2, 2, 2, 0, 3, 1, 1, 3, 0, 3, 1, 3, 0, 1, 0, 1, 3, 2, 0, 2, 2, 3, 2, 2, 3, 3, 2, 3,
     4 ] }
```

If you compare this path to the one for `/a/b/c` above, you'll see that the
first 64 2-bit characters match. This is because `/a/b` is a prefix of
`/a/b/c`.  Since this is the first entry, `seq` is 0.

Now we write `/a/c = hello` and get this node:

```js
{ key: '/a/c',
  value: 'hello',
  trie: [ , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , [ , , [ { feed: 0, seq: 0 } ] ] ],
  seq: 1,
  path: 
   [ 1, 2, 0, 1, 2, 0, 2, 2, 3, 0, 1, 2, 1, 3, 0, 3, 0, 0, 2, 1, 0, 2, 0, 0, 2, 0, 0, 3, 2, 1, 1, 2,
     0, 1, 1, 0, 1, 2, 3, 2, 2, 2, 0, 0, 3, 1, 2, 1, 3, 3, 3, 3, 3, 3, 0, 3, 3, 2, 3, 2, 3, 0, 1, 0,
     4 ] }
```

Its `seq` is 1, since the last was 0.  Also, this and the previous node have
the first 32 characters of their `path` in common (the prefix `/a`).

Notice though that `trie` is set. It's a long but sparse array. It has 35
entries, with the last one referencing the first node inserted (`a/b/`). Why?

(If it wasn't stored as a sparse array, you'd actually see 64 entries (the
length of the `path`). But since the other 29 entries are also empty, hyperdb
doesn't bother allocating them.)

If you visually compare this node's `path` with the previous node's `path`, how
many entries do they have in common? At which entry do the 2-bit numbers
diverge?

At the 35th entry.

What this is saying is "if the hash of the key you're looking for differs from
mine on the 35th entry, you want to travel to `{ feed: 0, seq: 0 }` to find the
node you're looking for.

This is how finding a node works, starting at any other node:

1. Compute the 2-bit hash sequence of the key you're after (e.g. `a/b`)
2. Lookup the newest entry in the feed.
3. Compare its `path` against the hash you just computed.
4. If you discover that the `path` and your hash match, then this is the node
   you're looking for!
5. Otherwise, once a 2-bit character from `path` and your hash disagree, note
   the index # where they differ and look up that value in the node's `trie`.
   Fetch that node at the given feed and sequence number, and go back to step 3.
   Repeat until you reach step 4 (match) or there is no entry in the node's trie
   for the key you're after (no match).

# Examples
[examples]: #examples

TODO: lookup a "deep" key

TODO: deleting a key

TODO: listing a prefix

# Drawbacks
[drawbacks]: #drawbacks

A backwards-incompatible change will have negative effects on the broader dat
ecosystem: clients will need to support both versions protocol for some time
(increasing maintenance burden), future clients may not interoperate with old
archives, etc. These downsides can partially be avoided by careful roll-out.

For the specific use case of Dat archives, HyperDB will trivially increase
metadata size (and thus disk and network consumption) for archives with few
files.


# Rationale and alternatives
[alternatives]: #alternatives

TODO: scaling issues with existing system; handling many entries

TODO: why multi-writer fields included here


# Dat migration logistics
[migration]: #migration

HyperDB is not backwards compatible with the existing hyperdrive metadata,
meaning dat clients may need to support both versions during a transition
period. This applies both to archives saved to disk (eg, in SLEEP) and to
archives received and published to peers over the network.

No changes to the Dat network wire protocol itself are necessary, only changes
to content passed over the protocol. The Dat `content` feed, containing raw
file data, is not impacted by HyperDB, only the contents of the `metadata`
feed.

Upgrading a Dat (hyperdrive) archive to HyperDB will necessitate creating a new
feed from scratch, meaning new public/private key pairs, and that public key
URL links will need to change.


# Unresolved questions
[unresolved]: #unresolved-questions

Details of keys to work out (before Draft status):

- must a prefix end in a trailing slash, or optional?
- can there be, eg, keys `/a/b` and `/a/b/c` at the same time?

There are implied "reasonable" limits on the size (in bytes) of both keys and
values, but they are not formally specified. Protobuf messages have a hard
specified limit of 2 GByte (due to 32-bit signed arthimetic), and most
implementations impose a (configurable) 64 MByte limit. Should this DEP impose
specific limits on key and value sizes? Would be good to decide before Draft
status.

Apart from leaving fields in the protobuf message specification, multi-writer
concerns are out of scope for this DEP.


# Changelog
[changelog]: #changelog

As of March 2018, @mafintosh is leading development of a hyperdb nodejs module
on [github](https://github.com/mafintosh/hyperdb), which is the basis for this
DEP.

- 2017-12-06: @noffle publishes `ARCHITECTURE.md` overview in the
  [hyperdb github repo][arch_md]
- 2018-02-XX: First complete draft submitted for review

[arch_md]: https://github.com/mafintosh/hyperdb/blob/master/ARCHITECTURE.md
