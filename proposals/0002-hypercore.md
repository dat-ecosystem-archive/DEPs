
Title: **DEP-0002: Hypercore**

Short Name: `0002-hypercore`

Type: Standard

Status: Draft (as of 2018-02-21)

Github PR: [Draft](https://github.com/datprotocol/DEPs/pull/4)

Authors: [Paul Frazee](https://github.com/pfrazee), [Mathias Buus](https://github.com/mafintosh)


# Summary
[summary]: #summary

Hypercore Feeds are the core mechanism used in Dat. They are binary append-only streams whose contents are cryptographically hashed and signed and therefore can be verified by anyone with access to the public key of the writer.

Dat uses two feeds, `content` and `metadata`. The `content` feed contains the files in your repository and `metadata` contains the metadata about the files including name, size, last modified time, etc. Dat replicates them both when synchronizing with another peer.


# Motivation
[motivation]: #motivation

Many datasets are shared online today using HTTP, which lacks built in support for version control or content addressing of data. This results in link rot and content drift as files are moved, updated or deleted. Dat solves this with a distributed, versioned file-sharing network that enables multiple untrusted devices to act as a single virtual host.

To ensure files are hosted correctly and can be referenced at different points in time, Dat needs a data structure which verifies the integrity of content and which retains a history of revisions. Notably, the data structure must:

 - Provide verification of file integrity using only the dataset identifier
 - Retain a full history of revisions to the dataset
 - Prevent dataset authors from altering the revision history after publishing to avoid a "split-brain" condition
 - Support efficient random and partial replication over the network


# Append-only Lists
[append-only-lists]: #append-only-lists

The Hypercore feed is an append-only list. The content of each list entry is an arbitrary blob of data. It can be replicated partially or fully over the network, and data can be received from multiple peers at once.

Internally, the Hypercore feed is represented by a signed merkle tree. The tree is identified on the network with a public key, which is then used to verify the signatures on received data. The tree is represented as a "Flat In-Order Tree."


## Flat In-Order Trees
[flat-in-order-trees]: #flat-in-order-trees

A Flat In-Order Tree is a simple way represent a binary tree as a list. It also allows you to identify every node of a binary tree with a numeric index. Both of these properties makes it useful in distributed applications to simplify wire protocols that uses tree structures.

Flat trees are described in [PPSP RFC 7574](https://datatracker.ietf.org/doc/rfc7574/?include_text=1) as "Bin numbers."

A sample flat tree spanning 4 blocks of data looks like this:

```
0
  1
2
    3
4
  5
6
```
      
The even numbered entries represent data blocks (leaf nodes) and odd numbered entries represent parent nodes that have two children.

The depth of a tree node can be calculated by counting the number of trailing 1s a node has in binary notation.

```
5 in binary = 101 (one trailing 1)
3 in binary = 011 (two trailing 1s)
4 in binary = 100 (zero trailing 1s)
```
      
1 is the parent of (0, 2), 5 is the parent of (4, 6), and 3 is the parent of (1, 5).

If the number of leaf nodes is a power of 2 the flat tree will only have a single root. Otherwise it'll have more than one. As an example here is a tree with 6 leafs:

```
0
  1
2
    3
4
  5
6

8
  9
10
```
      
The roots spanning all the above leafs are 3 an 9. Throughout this document we'll use following tree terminology:

 - `parent` - a node that has two children (odd numbered)
 - `leaf` - a node with no children (even numbered)
 - `sibling` - the other node with whom a node has a mutual parent
 - `uncle` - a parent's sibling


## Merkle Trees
[merkle-trees]: #merkle-trees

A merkle tree is a binary tree where every leaf is a hash of a data block and every parent is the hash of both of its children. Hypercore feeds are merkle trees encoded with "bin numbers" (see above).

Let's look at an example. A feed containing four values would be mapped to the even numbers 0, 2, 4, and 6.

```
chunk0 -> 0
chunk1 -> 2
chunk2 -> 4
chunk3 -> 6
```

Let `h(x)` be a hash function. Using flat-tree notation, the merkle tree spanning these data blocks looks like this:

```
0 = h(chunk0)
1 = h(0 + 2)
2 = h(chunk1)
3 = h(1 + 5)
4 = h(chunk2)
5 = h(4 + 6)
6 = h(chunk3)
```

In the resulting Merkle tree, the even and odd nodes store different information:

- Evens - List of data hashes [chunk0, chunk1, chunk2, ...]
- Odds - List of Merkle hashes (hashes of child even nodes) [hash0, hash1, hash2, ...]

In a merkle tree, the "root node" hashes the entire data set. In this example of 4 chunks, node 3 hashes the entire data set. Therefore we only need to trust node 3 to verify all data. As entries are added to a Hypercore feed, the "active" root node will change.

```
0
  1
2
    3 (root node)
4
  5
6
```

It is possible for the in-order Merkle tree to have multiple roots at once. For example, let's expand our example dataset to contain six items. This will result in two roots:

```
0
  1
2
    3 (root node)
4
  5
6

8
  9 (root node)
10
```

The nodes in this tree would be calculated as follows:

```
0 = h(chunk0)
1 = h(0 + 2)
2 = h(chunk1)
3 = h(1 + 5)
4 = h(chunk2)
5 = h(4 + 6)
6 = h(chunk3)

8 = h(chunk4)
9 = h(8 + 10)
10 = h(chunk5)
```

In the Hypercore feed, we only want one active root. Therefore, when there are multiple roots we hash all the roots together again. At most there will be `log2(number of data blocks)` root hashes to combine.

```
root = h(9 + 3)
```


## Root hash signatures
[root-hash-signatures]: #root-hash-signatures

Merkle trees are used to produce hashes which identify the content of a dataset. If the content of the dataset changes, the resulting hashes will change.

Hypercore feeds are internally represented by merkle trees, but act as lists which support the `append()` mutation. When this method is called, a new leaf node is added to the tree, generating a new root hash.

To provide a persistent identifer for a Hypercore feed, we generate an asymmetric keypair. The public key of the keypair is used as the identifier. Any time a new root hash is generated, it is signed using the private key. This signature is distributed with the root hash to provide verification of its integrity.


## Verifying received data
[verifying-received-data]: #verifying-received-data

To verify whether some received data belongs in a Hypercore feed, you must also receive a set of ancestor hashes which include a signed root hash. The signature of the root hash will first be verified to ensure it belongs to the Hypercore. The received data will then be hashed with the ancestor hashes in order to reproduce the root hash. If the calculated root hash matches the received signed root hash, then the data's correctness has been verified.

Let's look at an example for a feed containing four values (chunks). Our tree of hashes will look like this:

```
0
  1
2
    3 (root node)
4
  5
6
```

We want to receive and verify the data for 0 (chunk0). To accomplish this, we need to receive:

 - Chunk0
 - 2, the sibling hash
 - 5, the uncle hash
 - 3, the signed root hash

We will first verify the signature on 3. Then, we use the received data to recalculate 3:

```
0 = h(chunk0)
2 = (hash received)
1 = h(0 + 2)
5 = (hash received)
3 = h(1 + 5)
```

If our calculated 3 is equal to our received signed 3, then we know the chunk0 we received is valid.

Since we only need uncle hashes to verify the block, the number of hashes we need is at worst `log2(number-of-blocks)` and the roots of the merkle trees which has the same complexity.

Notice that all new signatures verify the entire dataset since they all sign a merkle tree that spans all data. If a signed update ever conflicts against previously-verified trees, suggesting a change in the history of data, the feed is considered "corrupt" and replication will stop. This serves to disincentivize changes to old data and avoids complexity around identifying the canonical history.


# Hash and signature functions
[hash-and-signature-functions]: #hash-and-signature-functions

The hash function used is `blake2b-256`. The signatures are `ed25519` with the `sha-512` hash function.

To protect against a ["second preimage attack"](https://en.m.wikipedia.org/wiki/Merkle_tree#Second_preimage_attack), hash functions have constants prepended to their inputs based on the type of data being hashed. These constants are:

 - `0x00` - Leaf
 - `0x01` - Parent
 - `0x02` - Root

Hashes will frequently include the sizes and indexes of their content in order to describe the structure of the tree, and not simply the data within the tree.

The cryptographic functions are defined as follows:

```js
// uint64be() encodes as a big-endian unsigned 64-bit int

function leaf_hash (data) {
  return blake2b([
    Buffer.from([0]),      // leaf constant, 0x00
    uint64be(data.length), // entry length in bytes
    data                   // entry data
  ])
}

function parent_hash (left, right) {
  if (left.index > right.index) {
    [left, right] = [right, left] // swap
  }

  return blake2b([
    Buffer.from([1]), // parent constant, 0x01
    uint64be(left.size + right.size),
    left.hash,
    right.hash
  ])
}

function root_hash (roots) {
  var buffers = new Array(3 * roots.length + 1)
  var j = 0

  buffers[j++] = Buffer.from([2]) // root constant, 0x02

  for (var i = 0; i < roots.length; i++) {
    var r = roots[i]
    buffers[j++] = r.hash
    buffers[j++] = uint64be(r.index)
    buffers[j++] = uint64be(r.size)
  }

  return blake2b(buffers)
}
```

# Parameters
[parameters]: #parameters

## Entry size
[parameters-entry-size]: #parameters-entry-size

The maximum size of a Hypercore feed entry is 8mb.

The Hypercore wire protocol applies an 10mb limit to message sizes. Accoringly, all entries on a Hypercore feed have an 8mb limit, to fit into a single message. Note that the 10mb/8mb limit is arbitrary and may be increased in the future.


# Drawbacks
[drawbacks]: #drawbacks

## The linear history requirement
[linear-history-requirement]: #linear-history-requirement

Hypercore assumes a linear history. It can not support branches in the history (multiple entries with the same sequence number) and will fail to replicate when a branch occurs. This means that applications must be extremely careful about ensuring correctness. Users can not copy private keys between devices or processes without strict coordination between them, otherwise they will generate branches and "corrupt" the hypercore.

## Private key risk
[private-key-risk]: #private-key-risk

Hypercore assumes that ownership of the private key is tantamount to authorship. If the private key is leaked, the author will lose control of the hypercore integrity. If the private key is lost, the hypercore will no longer be modifiable.


# Rationale and alternatives
[alternatives]: #alternatives

The Hypercore log is conceptually similar to Secure Scuttlebutt's log structure; both are designed to provide a single append-only history and to verify the integrity using only a public key identifier. However, Secure Scuttlebutt uses a Linked List structure with content-hash pointers to construct its log while Hypercore uses a Merkle Tree. This decision increases the amount of hashes computed and stored in Hypercore, but it enables more efficient partial replication of the dataset over the network as trees enable faster comparisons of dataset availability and verification of integrity. (Citation needed?)

IPFS is designed for immutable hash-addressed content, but it provides a mechanism for mutable content using public key addresses (IPNS). IPNS is still under development but some concepts are established. Its premise is much simpler than Hypercore's; rather than encoding the history in a standard form, IPNS simply signs and publishes a content-hash identifier under the public key, therefore creating a `pubkey -> hash` lookup. The referenced content may choose to encode a history, but it is not required and no constraints on branching is enforced. Compared to Hypercore, IPNS may be more user-friendly as it does not suffer from a catastrophic error if the history splits, therefore enabling users to share private keys freely between devices. This comes at the cost that history may be freely rewritten by the dataset author. Hypercore is also better suited to realtime streaming as it's possible to subscribe to optimistic broadcasts of updates.


# Unresolved questions
[unresolved]: #unresolved-questions

- Is there a potential "branch resolution" protocol which could remove the [linear history requirement](#the-linear-history-requirement) and therefore enable users to share private keys freely between their devices? Explaining the risks of branches to users is difficult. (This is being explored.)


# Changelog
[changelog]: #changelog

- 2018-02-19: First full draft of DEP-0002 submitted for review.
- 2018-02-21: DEP-0002 accepted as "Draft"
