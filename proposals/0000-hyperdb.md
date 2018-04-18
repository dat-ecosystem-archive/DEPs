
Title: **DEP-0000: Hyperdb**

Short Name: `0000-hyperdb`

Type: Standard

Status: Undefined (as of 2018-04-18)

Github PR: [Draft](https://github.com/datprotocol/DEPs/pull/3)

Authors:
[Bryan Newbold](https://github.com/bnewbold),
[Stephen Whitmore](https://github.com/noffle),
[Mathias Buus](https://github.com/mafintosh)


# Summary
[summary]: #summary

Hyperdb is an abstraction layer providing a general purpose distributed
key/value store over the hypercore protocol. It is an iteration on the
hyperdrive directory tree implementation, building top of the hypercore
append-only log abstraction layer. Keys are path-like strings (eg,
`/food/fruit/kiwi`), and values are arbitrary binary blobs (generally under a
megabyte).

Hyperdrive (used by the Dat application) is expected to be re-implemented on
top of hyperdb for improved performance with many files (eg, millions). The
hyperdrive API should be largely unchanged, but the `metadata` format will be
backwards-incompatible.


# Motivation
[motivation]: #motivation

Hyperdb is expected to drastically improve performance of dat clients when
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

*This section describes Hyperdb's interface and behavior in the abstract for
application programmers. It is not intended to be exact documentation of any
particular implementation (including the reference Javascript module).*

Hyperdb is structured to be used much like a traditional hierarchical
filesystem. A value can be written and read at locations like `/foo/bar/baz`,
and the API supports querying or tracking values at subpaths, like how watching
for changes on `/foo/bar` will report both changes to `/foo/bar/baz` and also
`/foo/bar/19`.

Lower-level details of the hypercore append-only log, disk serialization, and
networked synchronization features that Hyperdb builds on top of are not
described in detail here; see the [DEP repository][deps]. Multi-writer
hypercore semantics are also not discussed in this DEP.

[deps]: https://github.com/datprotocol/DEPs

A Hyperdb database instance can be represented by a single hypercore feed (or
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
[core_api]: #core_api

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
    db.put('/life/plant/bush/banana', '{"delicious": 103.4}')
    db.delete('/life/plant/bush/banana')
    db.put('/life/plant/tree/banana', '{"delicious": 103.4}')
    db.get('/life/animal/mammal/kitten')
    => {"cuteness": 500.3}
    db.list('/life/')
    => ['/life/animal/mammal/kitten', '/life/plant/tree/banana']


# Reference Documentation
[reference-documentation]: #reference-documentation

A hyperdb hypercore feed typically consists of a sequence of protobuf-encoded
messages of "Entry" type. Higher-level protocols may make exception to this,
for example by prepending an application-specific metadata message as the first
entry in the feed. There is sometimes a second "content" feed associated with
the primary hyperdb key/value feed, to store data that does not fit in the
(limited) `value` size constraint.

The sequence of entries includes an incremental index: the most recent entry in
the feed contains metadata pointers that can be followed to efficiently look up
any key in the database without needing to linear scan the entire history or
generate an independent index data structure. Implementations are, of course,
free to maintain their own index if they prefer.

The Entry protobuf message schema is:

    message Entry {
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

Some fields are are specific to the multi-writer features described in their
own DEP and mentioned only partially here. The fields common to both message
types are:

- `key`: UTF-8 key that this node describes. Leading and trailing forward
  slashes (`/`) are always striped before storing in protobuf.
- `value`: arbitrary byte array. A non-existant `value` entry indicates that
  this Entry indicates a deletion of the key; this is distinct from specifying
  an empty (zero-length) value.
- `trie`: a structured array of pointers to other Entry entries in the feed,
  used for navigating the tree of keys.
- `clock`: reserved for use in the forthcoming `multi-writer` standard An empty
  list is the safe (and expected) value for `clock` in single-writer use cases.
- `inflate`: a "pointer" (reference to a feed index number) to the most recent
  entry in the feed that has the optional `feeds` and `contentFeed` fields set.
  Not defined if `feeds` and `contentFeed` are defined.
- `feeds`: reserved for use with `multi-writer`. The safe single-writer value is
  to use the current feed's hypercore public key.
- `contentFeed`: for applications which require a parallel "content" hypercore
  feed for larger data, this field can be used to store the 32-byte public key
  for that feed.

For the case of a single-writer feed, not using multi-writer features, it is
sufficient to write a single Entry message as the first entry in the hypercore
feed, with `feeds` containing a single entry (a pointer to the current feed
itself), and `contendFeed` optionally set to a pointer to a paired contend
feed.

If *either* `feeds` *or* `contentFeed` are definined in an entry, *both* fields
must be defined, so the new message can be refered to via `inflated`. In this
case the entry is refered to as an "inflated entry".


## Path Hashing 
[path_hashing]: #path_hashing

Every key path has component-wise fixed-size hash representation that is used
by the trie. The concatenation of all path hashes yields a "path hash array"
for the entire key.  Note that analagously to a hash map data structure, there
can be more than one key (string) with the same key hash in the same database
with no problems: the hash points to a linked-list "bucket" of Entries, which
can be iterated over linearly to find the correct value. 

The path hash is represented by an array of bytes. Elements are 2-bit encoded
(values 0, 1, 2, 3), except for an optional terminating element which has value
4. Each path element consists of 32 values, representing a 64-bit hash of that
path element. For example, the key `/tree/willow` has two path segments (`tree`
and `willow`), and will be represented by a 65 element path hash array (two 32
element hashes plus a terminator).

The hash algorithm used is `SipHash-2-4`, with an 8-byte output and
16-byte key; the input is the UTF-8 encoded path string segment, without any
`/` separators or terminating null bytes. An implementation of this hash
algorithm is included in the libsodium library in the form of the
`crypto_shorthash()` function. A 16-byte "secret" key is required; for this use
case we use all zeros.

When converting the 8-bytehash to an array of 2-bit bytes, the ordering is
proceed byte-by-byte, and for each take the two lowest-value bits (aka, `hash &
0x3`) as byte index `0`, the next two bits (aka, `hash & 0xC`) as byte index
`1`, etc. When concatanating path hashes into a longer array, the first
("left-most") path element hash will correspond to byte indexes 0 through 31;
the terminator (`4`) will have the highest byte index.

For example, consider the key `/tree/willow`. `tree` has a hash `[0xAC, 0xDC,
0x05, 0x6C, 0x63, 0x9D, 0x87, 0xCA]`, which converts into the array:

```
[ 0, 3, 2, 2, 0, 3, 1, 3, 1, 1, 0, 0, 0, 3, 2, 1, 3, 0, 2, 1, 1, 3, 1, 2, 3, 1, 0, 2, 2, 2, 0, 3 ]
```


`willow` has a 64-bit hash `[0x72, 0x30, 0x34, 0x39, 0x35, 0xA8, 0x21, 0x44]`,
which converts into the array:

```
[ 2, 0, 3, 1, 0, 0, 3, 0, 0, 1, 3, 0, 1, 2, 3, 0, 1, 1, 3, 0, 0, 2, 2, 2, 1, 0, 2, 0, 0, 1, 0, 1 ]
```

These combine into the unified byte array with 65 elements:

```
[ 0, 3, 2, 2, 0, 3, 1, 3, 1, 1, 0, 0, 0, 3, 2, 1, 3, 0, 2, 1, 1, 3, 1, 2, 3, 1, 0, 2, 2, 2, 0, 3,
  2, 0, 3, 1, 0, 0, 3, 0, 0, 1, 3, 0, 1, 2, 3, 0, 1, 1, 3, 0, 0, 2, 2, 2, 1, 0, 2, 0, 0, 1, 0, 1,
  4 ]
```

As another example, the key `/a/b/c` converts into the 97-byte hash array:

```
[ 1, 2, 0, 1, 2, 0, 2, 2, 3, 0, 1, 2, 1, 3, 0, 3, 0, 0, 2, 1, 0, 2, 0, 0, 2, 0, 0, 3, 2, 1, 1, 2,
  0, 1, 2, 3, 2, 2, 2, 0, 3, 1, 1, 3, 0, 3, 1, 3, 0, 1, 0, 1, 3, 2, 0, 2, 2, 3, 2, 2, 3, 3, 2, 3,
  0, 1, 1, 0, 1, 2, 3, 2, 2, 2, 0, 0, 3, 1, 2, 1, 3, 3, 3, 3, 3, 3, 0, 3, 3, 2, 3, 2, 3, 0, 1, 0,
  4 ]
```

Note that "hash collisions" are rare with this hashing scheme, but are likely
to occur with large databases (millions of keys), and collision have been
generated as a proof of concept. Implementions should take care to properly
handle collisions by verifying keys and following bucket pointers (see the next
section).

An example hash collision (useful for testing; thanks to Github user
`dcposch`):

    /mpomeiehc
    /idgcmnmna

<!--

Generation code (javascript) for the above:

    var sodium = require('sodium-universal')
    var toBuffer = require('to-buffer')

    var KEY = Buffer.alloc(16)
    var OUT = Buffer.alloc(8)

    sodium.crypto_shorthash(OUT, toBuffer('tree'), KEY)
    console.log("tree: ", OUT)
    console.log(hash('tree', true))

    sodium.crypto_shorthash(OUT, toBuffer('willow'), KEY)
    console.log("willow: ", OUT)
    console.log(hash('willow', true))

    sodium.crypto_shorthash(OUT, toBuffer('a'), KEY)
    console.log("a: ", OUT)
    console.log(hash('a', true))

Then, to manually "expand" arrays in python3:

    hash_array = [0x72, 0x30, 0x34, 0x39, 0x35, 0xA8, 0x21, 0x44]
    path = []
    tmp = [(x & 0x3, (x >> 2) & 0x3, (x >> 4) & 0x3, (x >> 6) & 0x3) for x in hash_array]
    [path.extend(e) for e in tmp]
    path

-->


## Incremental Index Trie
[trie_index]: #trie_index

Each node stores a *prefix [trie](https://en.wikipedia.org/wiki/Trie)* that
can be used to look up other keys, or to list all keys with a given prefix.
This is stored under the `trie` field of the Entry protobuf message.

The trie effectively mirrors the path hash array. Each element in the `trie`
is called a "bucket". Each non-empty bucket points to the newest Entries which
have an identical path up to that specific prefix location; because the trie
has 4 "values" at each node, there can be pointers to up to 3 other values at a
given element in the trie array. Buckets can be empty if there are no nodes
with path hashes that differ for the first time the given bucket (aka, there
are no "branches" at this node in the trie). Only non-null elements will be
transmitted or stored on disk.

The data structure of the trie is a sparse array of pointers to other Entry
entries. Each pointer indicates a feed index and an entry index pointer, and is
associated with a 2-bit value; for the non-multi-writer case, the feed index is
always 0, so we consider only the entry index.

For a `trie` with `N` buckets, each may have zero or more pointers. Typically
there are a maximum of 3 pointers per bucket, corresponding to the 4 possible
values minus the current Entry's value, but in the event of hash collisions (in
the path array space) , there may be multiple pointers in the same bucket
corresponding to the same value.

To lookup a key in the database, the recipe is to:

1. Calculate the path hash array for the key you are looking for.
2. Select the most-recent ("latest") Entry for the feed.
3. Compare path hash arrays. If the paths match exactly, compare keys; they
   match, you have found the you were looking for! Check whether the `value` is
   defined or not; if not, this Entry represents that the key was deleted from
   the database.
4. If the paths match, but not the keys, look for a pointer in the last `trie`
   array index, and iterate from step #3 with the new Entry.
5. If the paths don't entirely match, find the first index at which the two
   arrays differ, and look up the corresponding element in this Entry's `trie`
   array. If the element is empty, or doesn't have a pointer corresponding to
   your 2-bit value, then your key does not exist in this hyperdb.
6. If the trie element is not empty, then follow that pointer to select the
   next Entry. Recursively repeat this process from step #3; you will be
   descending the `trie` in a search, and will either terminate in the Entry you
   are looking for, or find that the key is not defined in this hyperdb.

Similarly, to write a key to the database:

1. Calculate the path hash array for the key, and start with an empty `trie` of
   the same length; you will write to the `trie` array from the current index,
   which starts at 0.
2. Select the most-recent ("latest") Entry for the feed.
3. Compare path hash arrays. If the paths match exactly, and the keys match, then
   you are overwriting the current Entry, and can copy the "remainder" of it's
   `trie` up to your current `trie` index.
4. If the paths match, but not the keys, you are adding a new key to an
   existing hash bucket. Copy the `trie` and extend it to the full length. Add
   a pointer to the Entry with the same hash at the final array index.
5. If the paths don't entirely match, find the first index at which the two
   arrays differ. Copy all `trie` elements (empty or not) into the new `trie`
   for indicies between the "current index" and the "differing index".
6. Next look up the corresponding element in this Entry's `trie` array at the
   differing index. If this element is empty, then you have found the most
   similar Entry. Write a pointer to this node to the `trie` at
   the differing index, and you are done (all remaining `trie` elements are
   empty, and can be omitted).
7. If the differing tree element has a pointer (is not empty), then follow that
   pointer to select the next `Entry`. Recursively repeat this process from step
   #3.

To delete a value, follow the same procedure as adding a key, but write the
`Entry` without a `value` (in protobuf, this is distinct from having a `value`
field with zero bytes). Deletion nodes may persist in the database forever.


## Binary Trie Encoding
[trie_encoding]: #trie_encoding

The following scheme is used to encode trie data structures (sparse, indexed
arrays of pointers to entries) into a variable-length bytestring as the `trie`
field of an Entry protobuf message.

Consider a trie array with `N` buckets and `M` non-empty buckets (`0 <= M <=
N`). In the encoded field, there will be `M` concatanated bytestrings of the
form:

- trie index (varint), followed by...
- bucket bitfield (packed in a varint), encoding which of the 5 values (4
  values if the index is not modulo 32) at this node of the trie have pointers,
  followed by one or more...
- pointer sets, each referencing an entry at (feed index, entry index):
    - feed index (varint, with a extra "more pointers at this value" low bit,
      encoded as `feed_index << 1 | more_bit`)
    - entry index (varint)

In the common case for a small/sparse hyperdb, there will a small number of
non-empty buckets (small `M`), a usually a single `(feed index, entry index)`
pointer for those non-empty buckets. For a very large/dense hyperdb (millions
of key/value pairs), there will be many non-empty buckets (`M` approaching
`N`), and buckets may have up to the full 4 pointer sets. Even with millions of
entries, hash collisions will be very rare; in those cases there will be
multiple pointers in the same pointer set.

Consider an entry with path hash:

```
[ 1, 1, 0, 0, 3, 1, 2, 3, 3, 1, 1, 1, 2, 2, 1, 1, 1, 0, 2, 3, 3, 0, 1, 2, 1, 1, 2, 3, 0, 0, 2, 1,
  0, 2, 1, 0, 1, 1, 0, 1, 0, 1, 3, 1, 0, 0, 2, 3, 0, 1, 3, 2, 0, 3, 2, 0, 1, 0, 3, 2, 0, 2, 1, 1,
  4 ]
```

and trie:

```
[ , { val: 1, feed: 0, index: 1 } ]
```

In this case `N` is 64 (or you could count as 2 if you ignore trailing empty
entries) and `M` is 1. There will be a single bytestring chunk:

- index varint is `1` (second element in the trie array)
- bitfield is `0b0010` (varint 2): there is only a pointer set for value 1 (the second value)
- there is a single pointer in the pointer set, which is (`feed=0 << 1 | 0`,
  `index=1`), or (varint 2, varint 1)

Combined, the `trie` bytestring will be:

```
[0x01, 0x02, 0x02, 0x02]
```

For a more complex example, consider the same path hash, but the trie:

```
[ , { val: 1, feed: 0, index: 1; val: 2, feed: 5, index: 3; val: 3, feed: 6, index: 98 }, , , , , , , , , , , , , , , , , , , , , , , , , , , , , , ,
  , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , ,
  { val: 4, feed: 0, index, 23; val: 4, feed: 1, index: 17 ]
```

Now `M` is 2. The first bytestring chunk will be:

- index varint is `1` (second element in the trie array)
- bitfield is `0b01110` (varint 9): there are three pointer sets
- first pointer set is (`feed=0 << 1 | 0`, `index=1`) or (varint 2, varint 1)
- second pointer set is (`feed=5 << 1 | 0`, `index=3`) or (varint 10, varint 3)
- third pointer set is (`feed=6 << 1 | 0`, `index=98`) or (varint 12, varint 98)

the second bytestring chunk would be:

- index varint is `64` (65th element in the trie array; the terminating value)
- btifield is `0b10000` (varint 1); there is a single pointer set... but it
  contains a hash collision, so there are two pointers
- first pointer is (`feed=0` << 1 | 1`, `index=23`) or (varint 1, varint=23);
  note the `more` bit is set high!
- second pointer is (`feed=1 << 1 | 0`, `index=17`) or (varint 2, varint 17);
  note the `more` bit is low, as usual. In the extremely unlikely case of
  multiple collisions there could have been more pointers with `more` high
  preceeding this one.

The overall bytestring would be:

```
[0x01, 0x09, 0x02, 0x01, 0x0A, 0x03, 0x0C, 0x62, 0x40, 0x10, 0x01, 0x17, 0x02, 0x11]
```


# Examples
[examples]: #examples


## Simple Put and Get

Starting with an empty hyperdb `db`, if we `db.put('/a/b', '24')` we expect to
see a single `Entry` and index 0:

```
{ key: 'a/b',
  value: '24',
  trie:
   [ ] }
```

For reference, the path hash array for this key (index 0) is:

```
[ 1, 2, 0, 1, 2, 0, 2, 2, 3, 0, 1, 2, 1, 3, 0, 3, 0, 0, 2, 1, 0, 2, 0, 0, 2, 0, 0, 3, 2, 1, 1, 2,
  0, 1, 2, 3, 2, 2, 2, 0, 3, 1, 1, 3, 0, 3, 1, 3, 0, 1, 0, 1, 3, 2, 0, 2, 2, 3, 2, 2, 3, 3, 2, 3,
  4 ]
```

Note that the first 64 bytes in path match those of the `/a/b/c` example from
the [path hashing][path_hash] section, because the first two path components
are the same. Since this is the first entry, the entry index is 0.

Now we `db.put('/a/c', 'hello')` and expect a second Entry:

```
{ key: 'a/c',
  value: 'hello',
  trie:
   [ , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , 
     , , { element: 2, feed: 0, index: 0 } ] }
```

The path hash array for this key (index 1) is:

```
[ 1, 2, 0, 1, 2, 0, 2, 2, 3, 0, 1, 2, 1, 3, 0, 3, 0, 0, 2, 1, 0, 2, 0, 0, 2, 0, 0, 3, 2, 1, 1, 2,
  0, 1, 1, 0, 1, 2, 3, 2, 2, 2, 0, 0, 3, 1, 2, 1, 3, 3, 3, 3, 3, 3, 0, 3, 3, 2, 3, 2, 3, 0, 1, 0,
  4 ]
```

The first 32 characters of path are common with the first Entry (they share a
common prefix, `/a`).

`trie` is defined, but mostly sparse. The first 32 elements of common prefix
match the first Entry, and then two additional hash elements (`[0, 1]`) happen
to match as well; there is not a differing entry until index 34 (zero-indexed).
At this entry there is a reference pointing to the first Entry. An additional 29
trailing null entries have been trimmed in reduce metadata overhead.

Next we insert a third node with `db.put('/x/y', 'other')`, and get a third Entry:

```
{ key: 'x/y',
  value: 'other',
  trie:
   [ , { val: 1, feed: 0, index: 1 } ],
```

The path hash array for this key (index 2) is:

```
[ 1, 1, 0, 0, 3, 1, 2, 3, 3, 1, 1, 1, 2, 2, 1, 1, 1, 0, 2, 3, 3, 0, 1, 2, 1, 1, 2, 3, 0, 0, 2, 1,
  0, 2, 1, 0, 1, 1, 0, 1, 0, 1, 3, 1, 0, 0, 2, 3, 0, 1, 3, 2, 0, 3, 2, 0, 1, 0, 3, 2, 0, 2, 1, 1,
  4 ]
```

Consider the lookup-up process for `db.get('/a/b')` (which we expect to
successfully return `'24'`, as written in the first Entry). First we calculate
the path for the key `a/b`, which will be the same as the first Entry. Then we
take the "latest" Entry, with entry index 2. We compare the path hash arrays,
starting at the first element, and find the first difference at index 1 (`1 ==
1`, then `1 != 2`). We look at index 1 in the current Entry's `trie` and find a
pointer to entry index 1, so we fetch that Entry and recurse. Comparing path
hash arrays, we now get all the way to index 34 before there is a difference.
We again look in the `trie`, find a pointer to entry index 0, and fetch the
first Entry and recurse. Now the path elements match exactly; we have found the
Entry we are looking for, and it has an existant `value`, so we return the
`value`.

Consider a lookup for `db.get('/a/z')`; this key does not exist, so we expect
to return with "key not found". We calculate the path hash array for this key:

```
[ 1, 2, 0, 1, 2, 0, 2, 2, 3, 0, 1, 2, 1, 3, 0, 3, 0, 0, 2, 1, 0, 2, 0, 0, 2, 0, 0, 3, 2, 1, 1, 2,
  1, 2, 3, 0, 1, 0, 1, 1, 1, 1, 2, 1, 1, 1, 0, 1, 0, 3, 3, 2, 0, 3, 3, 1, 1, 0, 2, 1, 0, 1, 1, 2,
  4 ]
```

Similar to the first lookup, we start with entry index 2 and follow the pointer to
entry index 1. This time, when we compare path hash arrays, the first differing
entry is at array index `32`. There is no `trie` entry at this index, which
tells us that the key does not exist in the database.

## Listing a Prefix

Continuing with the state of the database above, we call `db.list('/a')` to
list all keys with the prefix `/a`.

We generate a path hash array for the key `/a`, without the terminating symbol
(`4`):

```
[ 1, 2, 0, 1, 2, 0, 2, 2, 3, 0, 1, 2, 1, 3, 0, 3, 0, 0, 2, 1, 0, 2, 0, 0, 2, 0, 0, 3, 2, 1, 1, 2 ]
```

Using the same process as a `get()` lookup, we find the first Entry that
entirely matches this prefix, which will be entry index 1. If we had failed to
find any Entry with a complete prefix match, then we would return an empty list
of matching keys.

Starting with the first prefix-matching node, we save that key as a match
(unless the Entry is a deletion), then select all `trie` pointers with an index
higher than the prefix length, and recursively inspect all pointed-to Entries.

## Deleting a Key

Continuing with the state of the database above, we call `db.delete('/a/c')` to
remove that key from the database.

The process is almost entirely the same as inserting a new Entry at that key,
except that the `value` field is undefined. The new Entry (at entry index 3)
is:

```
{ key: 'a/c',
  value: ,
  trie: [ , { val: 1, feed: 0, index: 2 }, , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , 
          , , { val: 1, feed: 0, index: 0 } ] }
```

The path hash array for this Entry (key) is:

```
[ 1, 2, 0, 1, 2, 0, 2, 2, 3, 0, 1, 2, 1, 3, 0, 3, 0, 0, 2, 1, 0, 2, 0, 0, 2, 0, 0, 3, 2, 1, 1, 2,
  0, 1, 1, 0, 1, 2, 3, 2, 2, 2, 0, 0, 3, 1, 2, 1, 3, 3, 3, 3, 3, 3, 0, 3, 3, 2, 3, 2, 3, 0, 1, 0,
  4 ]
```


# Drawbacks
[drawbacks]: #drawbacks

A backwards-incompatible change will have negative effects on the broader dat
ecosystem: clients will need to support both versions protocol for some time
(increasing maintenance burden), future clients may not interoperate with old
archives, etc. These downsides can partially be avoided by careful roll-out.

For the specific use case of Dat archives, hyperdb will trivially increase
metadata size (and thus disk and network consumption) for archives with few
files.


# Overhead and Scaling
[overhead]: #overhead

The metadata overhead for a single database entry varies based on the size of
the database. In a "heavy" case, considering a two-path-segment key with an
entirely saturated `trie` and `uint32`-sized feed and entry index pointers, and
ignoring multi-writer fields:

- `trie`: 4 * 2 * 64 bytes = 512 bytes
- total: 512 bytes

In a "light" case, with few `trie` entries and single-byte varint feed and
entry index pointers:

- `trie`: 2 * 2 * 4 bytes = 16 bytes
- total: 16

For a database with most keys having N path segments, the cost of a `get()`
scales with the number of entries M as `O(log(M))` with best case 1 lookup and
worst case `4 * 32 * N = 128 * N` lookups (for a saturated `trie`).

The cost of a `put()` or `delete()` is proportional to the cost of a `get()`.

The cost of a `list()` is linear (`O(M)`) in the number of matching entries,
plus the cost of a single `get()`.

The total metadata overhead for a database with M entries scales with `O(M
* log(M))`.


# Privacy and Security Considerations
[privacy]: #privacy

The basic key/value semantics of hyperdb (as discussed in this DEP, not
considering multi-writer changes) are not known to introduce new privacy issues
when compared with, eg, storing binary values at key-like paths using the
current ("legacy") hyperdrive system.

A malicious writer could cause trouble for readers, even readers who do not
trust the application-level contents of a feed. Implementions which may be
exposed to arbitrary feeds from unknown sources on the internet should take
care to the following scenarios: A malicious writer may be able to produce very
frequent key path hash collisions, which could degrade to linear performance. A
malicious writer could send broken trie structures that contain pointer loops,
duplicate pointers, or other invalid contents. A malicous writer could write
arbitrary data in value fields in an attempt to exploit de-serialization bugs.
A malicious writer could leverage non-printing unicode characters to create
database entries with user-indistinguishable names (keys).


# Rationale and alternatives
[alternatives]: #alternatives

A major motivator for hyperdb is to improve scaling performance with tens of
thousands through millions of files per directory in the existing hyperdrive
implementation. The current implementation requires the most recent node in a
directory to point to all other nodes in the directory. Even with pointer
compression, this requires on the order of `O(N^2)` bytes; the hyperdb
implementation scales with `O(N log(N))`.

The hyperdb specification (this document) is complicated by the inclusion of
new protobuf fields to support "multi-writer" features which are not described
here. The motivation to include these fields now to make only a single
backwards-incompatible schema change, and to make a second software-only change
in the future to enable support for these features. Schema and data format
changes are considered significantly more "expensive" for the community and
software ecosystem compared to software-only changes. Attempts have been made
in this specification to indicate the safe "single-writer-only" values to use
for these fields.


# Dat migration logistics
[migration]: #migration

Hyperdb is not backwards compatible with the existing hyperdrive metadata,
meaning dat clients may need to support both versions during a transition
period. This applies both to archives saved to disk (eg, in SLEEP) and to
archives received and published to peers over the network.

No changes to the Dat network wire protocol itself are necessary, only changes
to content passed over the protocol. The Dat `content` feed, containing raw
file data, is not impacted by hyperdb, only the contents of the `metadata`
feed.

Upgrading a Dat (hyperdrive) archive to hyperdb will necessitate creating a new
feed from scratch, meaning new public/private key pairs, and that public key
URL links will need to change.

Further logistical details are left to the forthcoming Multi-Writer DEP.


# Unresolved questions
[unresolved]: #unresolved-questions

Should we declare that `contendFeed` pointers *not* change over the history of
a feed? See also <https://github.com/datprotocol/DEPs/issues/13>

Need to think through deletion process with respect to listing a path prefix;
will previously deleted nodes be occulded, or potentially show up in list
results? Should be reviewed (by a non-author of this document) before accepted
as a Draft.

Can the deletion process (currently leaving "tombstone" entries in the `trie`
forever) be improved, such that these entries don't need to be iterated over?
mafintosh mentioned this might be in the works. Does this DEP need to "leave
room" for those changes, or should we call out the potential for future change?
Probably not, should only describe existing solutions. This can be resolved
after Draft.

Review (or prove?) the `O(log(M))` intuition for `get()` operations. This could
happen after Draft status.

There are implied "reasonable" limits on the size (in bytes) of both keys and
values, but they are not formally specified. Protobuf messages have a hard
specified limit of 2 GByte (due to 32-bit signed arthimetic), and most
implementations impose a (configurable) 64 MByte limit. Should this DEP impose
specific limits on key and value sizes? Would be good to decide before Draft
status.

Apart from leaving fields in the protobuf message specification, multi-writer
concerns are explicitly out of scope for this DEP.


# Changelog
[changelog]: #changelog

As of March 2018, Mathias Buus (@mafintosh) is leading development of a hyperdb
nodejs module on [github](https://github.com/mafintosh/hyperdb), which is the
basis for this DEP.

- 2017-12-06: Stephen Whitmore (@noffle) publishes `ARCHITECTURE.md` overview
  in the [hyperdb github repo][arch_md]
- 2018-03-04: First draft for review
- 2018-03-15: Hyperdb v3.0.0 is released
- 2018-04-18: This DEP submitted for Draft review.

[arch_md]: https://github.com/mafintosh/hyperdb/blob/master/ARCHITECTURE.md
