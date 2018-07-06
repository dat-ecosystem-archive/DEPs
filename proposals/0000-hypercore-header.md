
Title: **DEP-0000: Hypercore Header**

Short Name: `0000-hypercore-header`

Type: Standard

Status: Undefined (as of 2018-06-26)

Github PR: (add HTTPS link here after PR is opened)

Authors: [Paul Frazee](https://github.com/pfrazee), [Mathias Buus](https://github.com/mafintosh)


# Summary
[summary]: #summary

The Dat protocol supports a variety of data structures on top of the hypercore append-only logs, including key-value stores (hyperdb) and file archives (hyperdrive). In order for a program to read the data structure within a hypercore, it must know which structure it is reading. This DEP specifies a "header" entry which can be placed at the first entry of a hypercore in order to provide identifying information. This header also supports custom data for the data-structure, and may be expanded in future DEPs to provide additional standard data.


# Motivation
[motivation]: #motivation

At time of writing, Dat's standard data structures are being expanded upon (hyperdb) while existing data structures are being updated with breaking changes (hyperdrive). In order for these changes to be deployed smoothly to the network, it's important that the following needs be met:

 1. Programs should correctly identify the data structure type and version within a hypercore. This will enable the program to use the correct handling code (eg hyperdb, hyperdrive, hyperdrive-v2).
 2. Programs should abort processing a hypercore if they lack support for the type or version of the data structure.


# Usage Documentation
[usage-documentation]: #usage-documentation

The "header" is the first entry in a hypercore. It includes a `type` and an optional `extension` which can contain data-structure-specific data.

A program that is reading a hypercore will examine the `type` to determine how to process the hypercore.

```
function loadCorrectStructure (hypercore) {
  
  var header = parseHypercoreHeader(hypercore.readEntry(0))

  if (!header) {
    // no header present - treat as a hypercore
    return hypercore
  }

  switch (header.type) {
    case 'hyperdrive':
      return new HyperdriveV1(hypercore, header)
    case 'hyperdrive-v2':
      return new HyperdriveV2(hypercore, header)
    case 'hyperdb':
      return new HyperdbV1(hypercore, header)
    // ...
    default:
      // unknown type - treat as a hypercore
      return hypercore
  }

}
```

The `type` string should be unique to the data structure (refer to existing DEPs to avoid conflicts). When a breaking change is made to the data-structure, it should be given a new `type` string. For example, the `"hyperdrive"` type string will be updated to `"hyperdrive-v2"` for its second version.

The `header.extension` is an optional blob of bytes. It can be used by the data structure to store custom data. In `"hyperdrive"` (v1) it is used to specify the key of the content hypercore. It is also valid to encode a protobuf message into the extension field, and therefore it's simple to add additional fields.


# Reference Documentation
[reference-documentation]: #reference-documentation

The header message should be written as the first entry of the hypercore. It is not required that all hypercores possess a header.

The header message has the following protobuf schema:

```protobuf
message HypercoreHeader {
  required string type = 1;
  optional bytes extension = 2;
}
```

Data structures should not add additional fields to the `HypercoreHeader`. Any additional fields should be ignored. Data-structure-specific data can be stored in the `extension`. This is easy to do with protobuf's nested messages. For example:

```protobuf
message MyCustomHeader {
  message AdditionalData {
    required string foo = 1;
    required uint64 bar = 2;
  }
  required string type = 1;
  optional AdditionalData extension = 2;
}
```


# Rationale and alternatives
[alternatives]: #alternatives

This standard is backwards compatible with v1 of hyperdrive, which used an `Index` message as the first entry in the metadata hypercore.

An alternative approach is to simply attempt to process a given hypercore with all known message encodings. However, this is time-consuming and error-prone, and may not accurately capture the *version* of a data-structure.


# Changelog
[changelog]: #changelog

- 2018-06-26: First complete draft submitted for review

