
Title: **DEP-0000: Hyperdrive File Hashes**

Short Name: `0000-hyperdrive-hashes`

Type: Standard

Status: Undefined (as of YYYY-MM-DD)

Github PR: [Discussion](https://github.com/datprotocol/DEPs/pull/12)

Authors: [Bryan Newbold](https://github.com/bnewbold)


# Summary
[summary]: #summary

Full-file hashes are optionally included in hyperdrive metadata to complement
the existing cryptographic-strength hashing of sub-file chunks. Multiple
popular hash algorithms can be included at the same time.


# Motivation
[motivation]: #motivation

Naming, discovering, and cataloging data "by content" (aka, by a fixed-size
hashes of the data) is a powerful pattern for robust distributed systems. Dat
is one among several such systems. Unfortunately, interoperability between or
layering such systems on top of each other is difficult because each tends to
adopt it's own hashing norms and formats. Design variances can include hash
algorithm selection, hash configuration, salting, data chunking, and
intermediate Merkle tree data formats.

As a concrete example, the sha1sum command-line tool, the bittorrent P2P
protocol and the git code versioning software both use the SHA-1 algorithm to
hash file contents. However, one can not use the simple `sha1sum` hash of a
given file to check whether that file is the same as referenced in either a
bittorrent `.torrent` file or from git metadata, because each calculate the
hash in different ways. Bittorrent combines all files in the torrent into a
single stream, then splits into a fixed number of chunks and hashes those
separately; the chunk boundaries usually do not correspond to individual files.
git prepends the size of the file (in bytes) as a fixed header before hashing
and storing the file as a "blob". This makes comparison or interoperability
between these systems impossible without having either a universal cross-hash
table (infeasible to build in the general sense) or without having the full
file contents on-hand to compare or re-hash in all three formats.

The design decisions to adopt hash variants are usually well-founded, motivated
by security concerns (such as pre-image attacks), efficiency, and
implementation concerns.

By adding simple full-file hashes of files as optional complementary metadata
in our distributed data systems, we can make interoperability and powerful
efficiency gains possible.

For example, a large collection of files could be stored in a simple format on
disk, indexed by a popular hash format. Gateway clients to several P2P networks
could make the same files accessible by storing metadata (relatively small)
separately for each network, but accessing the file contents from the shared
store by a common hash.

In the case of Dat, a particular efficiency of this use case would be enabling
fast de-duplication of file storage between multiple Dat archives on a
full-file level, instead of at the chunk-level (which would be sensitive to
changes in chunking algorithm).


# Usage Documentation
[usage-documentation]: #usage-documentation

Implementations would include hashes as file-level metadata along with existing
"stat" fields.

Existing API methods would include options to control generation of hashes (and
which types) when creating a new drive or adding files.


# Reference Documentation
[reference-documentation]: #reference-documentation

Hashes would be stored as additional fields in hyperdrive's existing `Stat`
protobuf message, with the following structure:

```protobuf
message Stat {
  message ExtraHash {
    required uint32 type = 1;
    required bytes value = 2;
  }
  required uint32 mode = 1;
  optional uint32 uid = 2;
  optional uint32 gid = 3;
  optional uint64 size = 4;
  optional uint64 blocks = 5;
  optional uint64 offset = 6;
  optional uint64 byteOffset = 7;
  optional uint64 mtime = 8;
  optional uint64 ctime = 9;
  repeated ExtraHash hashes = 10;
}
```

`type` is a number representing the hash algorithm, and `value` is the
bytestring of the hash output itself. The length of the hash digest (in bytes)
is available from protobuf metadata for the value. This scheme, and the `type`
value table, is intended to be interoperable with the [multihash][multihash]
scheme from the IPFS community.

A subset of the multihash hash digest table includes:

```
    md5         0x00D5
    sha1        0x0011
    sha2-256    0x0012
    sha2-512    0x0013
    blake2b-256 0xB220
```

Multiple hashes would be calculated in parallel with the existing
chunking/hashing process, in a streaming fashion. Final hashes would be
calculated when the chunking is complete, and included in the `Stat` metadata.

For 2018, recommended default full-file hash functions to include are `SHA1`
(for popularity and interoperability) and `blake2b-256` (already used in other
parts of the Dat protocol stack).

[multihash]: https://multiformats.io/multihash/


# Drawbacks
[drawbacks]: #drawbacks

The metadata storage overhead (on a per-file basis) should be minimal, but the
additional computational resources to hash a large file multiple times are
non-trivial on machines with a single (or few) cores, even when computed in a
parallel/streaming format.


# Security and Privacy Concerns
[privacy]: #privacy

Additional optional fields may leak additional bits of user-specific
configuration metadata, analogous to the "[evercookie][]" and
"[panopticlick][]" browser fingerprinting issues.

[evercookie]: https://en.wikipedia.org/wiki/Evercookie
[panopticlick]: https://panopticlick.eff.org/


# Rationale and alternatives
[alternatives]: #alternatives

Users wanting this metadata could instead maintain a manifest file (mapping
paths to hashes) inside the Dat archive itself. The Dat client could support
this with a special mode or flag. One downside of this is that for large
archives, the file would need to be updated and duplicated for every new or
modified file.


# Unresolved questions
[unresolved]: #unresolved-questions

What does the user-facing API look like, specifically?

Should we allow non-standard hashes, like the git "hash", or higher-level
references like (single-file) bittorrent magnet links or IPFS file references?

Modifying a small part of a large file would require re-hashing the entire
file, which is slow. Should we skip including the updated hashes in this case?
Currently mitigated by the fact that we duplicate the entire file when recoding
changes or additions.


# Changelog
[changelog]: #changelog

- 2018-03-17: First draft for comment.

