
Title: **DEP-0000: Hyperdrive**

Short Name: `0000-hyperdrive`

Type: Standard

Status: Undefined (as of 2018-02-05)

Github PR: (add HTTPS link here after PR is opened)

Authors: [Paul Frazee](https://github.com/pfrazee)


# Summary
[summary]: #summary

Hyperdrive archives are the top-level abstraction in Dat. They provide an interface for listing, reading, and writing files and folders. Each Hyperdrive is constructed using two Hypercore registers: one for representing the file metadata, and the other for representing the file content.

As an example, consider a folder with two files:

```
bat.jpg
cat.jpg
```

These files are added to a Hyperdrive archive by splitting them into chunks and constructing Hypercore registers representing the chunks and filesystem metadata.

Let's assume `bat.jpg` and `cat.jpg` both produce three chunks, each around 64KB. Here we will show a pseudo-representation for the purposes of illustrating the replication process. The six chunks get sorted into a list like this:

```
bat-1
bat-2
bat-3
cat-1
cat-2
cat-3
```

Via Hypercore, these chunks are hashed, and the hashes are arranged into a Merkle tree (the content register):

```
0          - hash(bat-1)
    1      - hash(0 + 2)
2          - hash(bat-2)
        3  - hash(1 + 5)
4          - hash(bat-3)
    5      - hash(4 + 6)
6          - hash(cat-1)
8          - hash(cat-2)
    9      - hash(8 + 10)
10         - hash(cat-3)
```

This tree is for the hashes of the contents of the photos. There is also a second Hypercore register that Hyperdrive generates that represents the list of files and their metadata and looks something like this (the metadata register):

```
0 - hash({contentRegister: '9e29d624...'})
  1 - hash(0 + 2)
2 - hash({"bat.jpg", offset: 0, length: 3})
4 - hash({"cat.jpg", offset: 3, length: 3})
```

The first entry in this feed is a special metadata entry that tells Dat the address of the second feed, the content register. The remaining two entries declare the existence of the two files, and indicate where to find their data in the content register.


# File archives
[file-archives]: #file-archives

TODO


## File metadata
[file-metadata]: #file-metadata

Dat tries as much as possible to act as a one-to-one mirror of the state of a folder and all its contents. When importing files, Dat uses a sorted, depth-first recursion to list all the files in the tree. For each file it finds, it grabs the filesystem metadata (filename, Stat object, etc) and appends the data to the metadata register.


## Append-tree data structure
[append-tree-data-structure]: #append-tree-data-structure

"Append-tree" is a tree data structure modeled on top of an append-only log (the Hypercore register). The data structure stores a small index for every entry in the log so that no external indexing is required to model the tree. It also provides fast lookups on sparsely replicated logs.

TODO- describe


# Hypercore entry schemas
[hypercore-entry-schemas]: #hypercore-entry-schemas

Hyperdrive encodes its data to its Hypercore registers using Google's [protobuf](https://developers.google.com/protocol-buffers/) encoding.


## Index
[hypercore-entry-schema-index]: #hypercore-entry-schema-index

The schema of the first entry written in every metadata Hypercore register. The `type` field will contain the string `"hyperdrive"`. The `content` field will contain the public key of the content Hypercore register.

```
message Index {
  required string type = 1;
  optional bytes content = 2;
}
```


## Node
[hypercore-entry-schema-node]: #hypercore-entry-schema-node

This entry schema encodes a file in the metadata Hypercore register. The `name` field is a string providing the file-path. The `value` field is a Stat object, using the `Stat` encoding (below). The `paths` field is an index used to optimize lookups (TODO how?).

```
message Node {
  required string name = 1;
  optional bytes value = 2;
  optional bytes paths = 3;
}
```


## Stat
[hypercore-entry-schema-stat]: #hypercore-entry-schema-stat

This schema encodes the "Stat" of a file in the metadata Hypercore register. TODO- describe fields.

```
message Stat {
  required uint32 mode = 1;
  optional uint32 uid = 2;
  optional uint32 gid = 3;
  optional uint64 size = 4;
  optional uint64 blocks = 5;
  optional uint64 offset = 6;
  optional uint64 byteOffset = 7;
  optional uint64 mtime = 8;
  optional uint64 ctime = 9;
}
```


# Specifications and parameters
[specifications-and-parameters]: #specifications-and-parameters

All file paths are converted to Unix-style with a forward-slash separator.


# Drawbacks
[drawbacks]: #drawbacks

TODO


# Rationale and alternatives
[alternatives]: #alternatives

TODO


# Unresolved questions
[unresolved]: #unresolved-questions

TODO


# Changelog
[changelog]: #changelog

A brief statemnt about current status can go here, follow by a list of dates
when the status line of this DEP changed (in most-recent-last order).

- YYYY-MM-DD: First complete draft submitted for review
