
Title: **DEP-0000: HyperDB**

Short Name: `0000-hyperdb`

Type: Standard

Status: Undefined (as of 2018-01-XXX)

Github PR: (add HTTPS link here after PR is opened)

Authors: noffle (Stephen Whitmore), bnewbold (Bryan Newbold)


# Summary
[summary]: #summary

HyperDB is a new abstraction layer providing a general purpose distributed
key/value store over the Dat protocol. It is an iteration on the hyperdrive
directory tree implementation, providing a general purpose key/value store on
top of the hypercore append-only log abstraction layer. Keys are path-like
strings (eg, `/food/fruit/kiwi`), and values are arbitrary binary blobs (with
expected size of under a megabyte).

hyperdrive is expected to be re-implemented on top of HyperDB for improved
performance with many (millions) of files. The hyperdrive API should be largely
unchanged, but the `metadata` format will be backwards-incompatible.


# Motivation
[motivation]: #motivation

HyperDB is expected to drastically improve performance of dat clients when
working with archives containing tens of thousands of files in single
directories. This is a real-world bottleneck for several current users, with
basic local actions such as adding a directory taking an unacceptably long time
to complete.

A secondary benefit is to refactor the trie-structured key/value API out of
hyperdrive, allowing third party code to build on this abstraction layer.


# Usage Documentation
[usage-documentation]: #usage-documentation

HyperDB is structured to be used much like a traditional hierarchical
filesystem. A value can be written and read at locations like `/foo/bar/baz`,
and the API supports querying or tracking values at subpaths, like how watching
for changes on `/foo/bar` will report both changes to `/foo/bar/baz` and also
`/foo/bar/19`.

## New API

`add(key, value)`

`get(key)`

`delete(key)`

# Reference Documentation
[reference-documentation]: #reference-documentation

## Set of append-only logs (feeds)

A HyperDB is fundamentally a set of
[hypercore](https://github.com/mafintosh/hypercore)s. A *hypercore* is a secure
append-only log that is identified by a public key, and can only be written to
by the holder of the corresponding private key.

Each entry in a hypercore has a *sequence number*, that increments by 1 with
each write, starting at 0 (`seq=0`).

HyperDB builds its hierarchical key-value store on top of these hypercore
feeds, and also provides facilities for authorization, and replication of those
member hypercores.

## Incremental index

HyperDB builds an *incremental index* with every new key/value pairs ("nodes")
written. This means a separate data structure doesn't need to be maintained
elsewhere for fast writes and lookups: each node written has enough information
to look up any other key quickly and otherwise navigate the database.

Each node stores the following basic information:

- `key`: the key that is being created or modified. e.g. `/home/sww/dev.md`
- `value`: the value stored at that key.
- `seq`: the sequence number of this entry in the hypercore. 0 is the first, 1
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

# Migration logistics
[migration]: #migration

HyperDB is not backwards compatible with the existing hyperdrive
implementation, meaning dat clients will need to support multiple on-disk
representations during a transition period.

a new abstraction layer between hypercore (replicated append-only
logs) and hyperdrive (versioned file system abstraction).  HyperDB provides an
efficient key/value database API, with path-like strings as keys and arbitrary
binary data (up to a reasonable chunk size) as values. HyperDB will require
breaking changes to dat clients, but will not require changes to the network
wire protocol.

# Changelog
[changelog]: #changelog

As of January 2018, @mafintosh is leading development of a hyperdb nodejs
module on [github](https://github.com/mafintosh/hyperdb), which is the basis
for this DEP.

- 2017-12-06: @noffle publishes `ARCHITECTURE.md` overview in the
  [hyperdb github repo][arch_md]
- 2018-01-XXX: First complete draft submitted for review

[arch_md]: https://github.com/mafintosh/hyperdb/blob/master/ARCHITECTURE.md
