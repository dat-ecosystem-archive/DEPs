# 2018-12-27 Hypercore SLEEP Headers DEP

Title: DEP-0000: Hypercore SLEEP Headers

Short Name: `0000-sleep-headers`

Type: Standard

Status: Undefined (as of 2018-12-27)

Github PR: (add HTTPS link here after PR is opened)

Authors: Yoshua Wuyts <yoshuawuyts@gmail.com>

# Summary

This documents the SLEEP headers that are part of Hypercore protocol.

# Motivation

SLEEP headers in the Hypercore protocol contain metadata explaining how a feed
is constructed, and how to interpret the data in it.

These headers have been described from a high-level in the past, but have always
left some room for interpretation. This DEP creates a specification on how to
construct and parse Hypercore SLEEP headers.

# Usage Documentation

The acronym SLEEP is a slumber related pun on REST and stands for Syncable
Ledger of Exact Events Protocol. The Syncable part refers to how SLEEP files are
append-only in nature, meaning they grow over time and new updates can be
subscribed to as a realtime feed of events through Hypercore.

SLEEP files are split up into a Header and a Body part. This DEP only concerns
itself with the SLEEP headers.

SLEEP headers are located at the start of files. They contain metadata about the
rest of the file, such as how they’re formatted, which algorithms they use, and
how large the individual entries are. Not each file in a SLEEP archive has a
Hypercore header.

Hypercore files with SLEEP headers are:

- bitfield
- signatures
- (Merkle) tree

Hypercore files without SLEEP headers are:

- public key
- secret key

# Reference Documentation

## Header Layout

Each SLEEP header is 32 bytes. Different file types have different header
configurations. This section describes the shared layout of the headers.

    <32 byte header>
      <3 byte magic string: 0x050257>
      <1 byte header type>
      <1 byte version number: 0>
      <2 byte entry size>
      <1 byte algorithm name length prefix>
      <variable byte algorithm name>
      <variable byte padding>

### 3 byte magic string
The first 3 bytes in the header are “magic bytes” that serve to identify files
as SLEEP formatted. These numbers correspond to Mathias Buus’s birthday in hex
notation (05/02/87).

### 1 byte header type
The 4th byte determines what kind of header file this is. Currently 3 header
types supported:

- bitfield (`0x00`)
- signatures (`0x01`)
- (Merkle) tree (`0x02`).

### 1 byte version number
The 5th byte is used to determine the protocol version. Currently only one
version of the protocol exists, so this number is always set to `0x00`.

### 2 byte entry size (u16BE)
Each entry in the SLEEP file’s body has a fixed size. The size of these entries
is determined by the entry size value. This value has to be read as a 16-bit
Big-Endian value.

### 1 byte algorithm name prefix
SLEEP files can be encoded using different algorithms. The algorithm names are
encoded as strings, and not as numbers. The 1 byte algorithm name prefix exists
to signal how long the algorithm name string will be.

### variable byte algorithm name
The algorithm name determines how SLEEP files are encoded. The length of the
algorithm name is variable length, and encoded as the 1 byte algorithm name
prefix.

Currently 3 possible variants exist:

- `BLAKE2b` (7 bytes)
- `Ed25519` (7 bytes)
- None (0 bytes)

### variable byte padding
Each SLEEP header should be a total of 32 bytes. If there is not enough data to
fill 32 bytes, the remainder of the header should be filled with zeroes
(`0x00`).

However, for forward compatibility reasons, parsers should not expect that the
remainder of the header is filled with zeroes. As long as the protocol version
is understood by the parser, it should be safe to ignore the remainder of the
data in the header, regardless of its encoding.

## Bitfield Layout

This describes the header layout of SLEEP headers in the bitfield configuration


    <32 byte header>
      <3 byte magic string: 0x050257>
      <1 byte header type: 0x00>
      <1 byte version number: 0>
      <2 byte entry size: 3328>
      <1 byte algorithm name length prefix: 0>
      <0 byte algorithm name>
      <24 byte padding>

### 1 byte header type: 0x00
Bitfields should have the header type of `0x00`.

### 2 byte entry size (u16BE): 3328
Each entry in the bitfield’s body is `3328` bytes. This is the combined leng
of Hypercore’s 3 individual bitfields:

- data bitfield: `1024` bytes
- tree bitfield: `2048` bytes
- index bitfield: `256` bytes

Together these add up to `3328`.

### 1 byte algorithm name prefix: 0
Bitfields make use of a Run-Length Encoding (RLE) algorithm to encode their
contents. However this is only for compression purposes, and not included as
part of the algorithm name.

### variable byte algorithm name
No algorithm name is provided.

### variable byte padding: 24
`24` zeroes are appended to the end of the header to create a total of 32 bytes.

## Signatures Layout

This describes the header layout of SLEEP headers in the signatures
configuration

    <32 byte header>
      <3 byte magic string: 0x050257>
      <1 byte header type: 0x01>
      <1 byte version number: 0>
      <2 byte entry size: 64>
      <1 byte algorithm name length prefix: 7>
      <7 byte algorithm name: Ed25519>
      <21 byte padding>

### 1 byte header type: 0x01
Signatures should have the header type of `0x01`.

### 2 byte entry size (u16BE): 64
Each signature in the signatures file is `64` bytes.

### 1 byte algorithm name prefix: 7
The string `Ed25519` is 7 characters long.

### variable byte algorithm name: Ed25519
Signatures are created using the `Ed25519` encryption scheme.

### variable byte padding: 21
`21` zeroes are appended to the end of the header to create a total of 32 bytes.

## Merkle Tree Layout

This describes the header layout of SLEEP headers in the tree configuration

    <32 byte header>
      <3 byte magic string: 0x050257>
      <1 byte header type: 0x02>
      <1 byte version number: 0>
      <2 byte entry size: 40>
      <1 byte algorithm name length prefix: 7>
      <7 byte algorithm name: BLAKE2b>
      <21 byte padding>

### 1 byte header type: 0x02
Merkle Trees should have the header type of `0x02`.

### 2 byte entry size (u16BE): 40
Each entry in the tree file is `40` bytes. The first 32 bytes is the hash. The
next 8 bytes is the byte size of the spanning tree.

### 1 byte algorithm name prefix: 7
The string `BLAKE2b` is 7 characters long.

### variable byte algorithm name: BLAKE2b
Merkle Tree entries are created using the `BLAKE2b` hashing scheme.

### variable byte padding: 21
`21` zeroes are appended to the end of the header to create a total of 32 bytes.

# Drawbacks

None.

# Rationale and alternatives
## Fail parsing if padding contains anything other than zeros

Parsers could choose to fail parsing if anything other than zeros was used to
pad the end of the header. However, this would break forward compatibility with
any header extensions.

Because the header has a versioning field, any non-backward compatible changes
to the header should result in a version increase. So as long as header versions
line up, any additions to the header should be additive, and not break the
existing parsing.

# Unresolved questions
- There should be a follow-up DEP to specify the body portion of SLEEP files.

# Changelog

A brief statement about current status can go here, follow by a list of dates
when the status line of this DEP changed (in most-recent-last order).

- 2018-12-28: First complete draft submitted for review
