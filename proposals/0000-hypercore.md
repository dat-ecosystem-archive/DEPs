
Title: **DEP-0000: Hypercore**

Short Name: `0000-hypercore`

Type: Standard

Status: Undefined (as of 2018-02-05)

Github PR: (add HTTPS link here after PR is opened)

Authors: [Paul Frazee](https://github.com/pfrazee)


# Summary
[summary]: #summary

Hypercore Registers are the core mechanism used in Dat. They are binary append-only streams whose contents are cryptographically hashed and signed and therefore can be verified by anyone with access to the public key of the writer. They are an implementation of the concept known as a register, a digital ledger you can trust.

Dat uses two registers, `content` and `metadata`. The `content` register contains the files in your repository and `metadata` contains the metadata about the files including name, size, last modified time, etc. Dat replicates them both when synchronizing with another peer.


# Motivation
[motivation]: #motivation

Many datasets are shared online today using HTTP and FTP, which lack built in support for version control or content addressing of data. This results in link rot and content drift as files are moved, updated or deleted. Dat solves this with a distributed, versioned file-sharing network which enables multiple devices to act in concert as a single virtual host.

To ensure files are hosted correctly by untrusted devices, Dat needs a data structure which verifies the integrity of content and which retains a history of revisions. Notably, the data structure must:

 - Provide verification of file integrity using only the dataset identifier
 - Retain a full history of revisions to the dataset
 - Prevent dataset authors from altering the revision history after publishing to avoid a "split-brain" condition
 - Support efficient random and partial replication over the network



## Merkle Trees
[merkle-trees]: #merkle-trees

Registers in Dat use a specific method of encoding a Merkle tree where hashes are positioned by a scheme called binary in-order interval numbering or just "bin" numbering. This is just a specific, deterministic way of laying out the nodes in a tree. For example a tree with 7 nodes will always be arranged like this:

```
0
  1
2
    3
4
  5
6
```

In Dat, the hashes of the chunks of files are always even numbers, at the wide end of the tree. So the above tree had four original values that become the even numbers:

```
chunk0 -> 0
chunk1 -> 2
chunk2 -> 4
chunk3 -> 6
```

In the resulting Merkle tree, the even and odd nodes store different information:

- Evens - List of data hashes [chunk0, chunk1, chunk2, ...]
- Odds - List of Merkle hashes (hashes of child even nodes) [hash0, hash1, hash2, ...]

These two lists get interleaved into a single register such that the indexes (position) in the register are the same as the bin numbers from the Merkle tree.

All odd hashes are derived by hashing the two child nodes, e.g. given hash0 is `hash(chunk0)` and hash2 is `hash(chunk1)`, hash1 is `hash(hash0 + hash2)`.

For example a register with two data entries would look something like this (pseudocode):

```
0. hash(chunk0)
1. hash(hash(chunk0) + hash(chunk1))
2. hash(chunk1)
```

It is possible for the in-order Merkle tree to have multiple roots at once. A root is defined as a parent node with a full set of child node slots filled below it.

For example, this tree has 2 roots (1 and 4)

```
0
  1
2

4
```

This tree has one root (3):

```
0
  1
2
    3
4
  5
6
```

This one has one root (1):

```
0
  1
2
```


## Replication Example

This section describes in high level the replication flow of a Dat. Note that the low level details are available by reading the SLEEP section below. For the sake of illustrating how this works in practice in a networked replication scenario, consider a folder with two files:

```
bat.jpg
cat.jpg
```

To send these files to another machine using Dat, you would first add them to a Dat repository by splitting them into chunks and constructing SLEEP files representing the chunks and filesystem metadata.

Let's assume `bat.jpg` and `cat.jpg` both produce three chunks, each around 64KB. Dat stores in a representation called SLEEP, but here we will show a pseudo-representation for the purposes of illustrating the replication process. The six chunks get sorted into a list like this:

```
bat-1
bat-2
bat-3
cat-1
cat-2
cat-3
```

These chunks then each get hashed, and the hashes get arranged into a Merkle tree (the content register):

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

Next we calculate the root hashes of our tree, in this case 3 and 9. We then hash them together, and cryptographically sign the hash. This signed hash now can be used to verify all nodes in the tree, and the signature proves it was produced by us, the holder of the private key for this Dat.

This tree is for the hashes of the contents of the photos. There is also a second Merkle tree that Dat generates that represents the list of files and their metadata and looks something like this (the metadata register):

```
0 - hash({contentRegister: '9e29d624...'})
  1 - hash(0 + 2)
2 - hash({"bat.jpg", first: 0, length: 3})
4 - hash({"cat.jpg", first: 3, length: 3})
```

The first entry in this feed is a special metadata entry that tells Dat the address of the second feed (the content register). Note that node 3 is not included yet, because 3 is the hash of `1 + 5`, but 5 does not exist yet, so will be written at a later update.

Now we're ready to send our metadata to the other peer. The first message is a `Register` message with the key that was shared for this Dat. Let's call ourselves Alice and the other peer Bob. Alice sends Bob a `Want` message that declares they want all nodes in the file list (the metadata register). Bob replies with a single `Have` message that indicates he has 2 nodes of data. Alice sends three `Request` messages, one for each leaf node (`0, 2, 4`). Bob sends back three `Data` messages. The first `Data` message contains the content register key, the hash of the sibling, in this case node `2`, the hash of the uncle root `4`, as well as a signature for the root hashes (in this case `1, 4`). Alice verifies the integrity of this first `Data` message by hashing the metadata received for the content register metadata to produce the hash for node `0`. They then hash the hash `0` with the hash `2` that was included to reproduce hash `1`, and hashes their `1` with the value for `4` they received, which they can use the received signature to verify it was the same data. When the next `Data` message is received, a similar process is performed to verify the content.

Now Alice has the full list of files in the Dat, but decides they only want to download `cat.png`. Alice knows they want blocks 3 through 6 from the content register. First Alice sends another `Register` message with the content key to open a new replication channel over the connection. Then Alice sends three `Request` messages, one for each of blocks `4, 5, 6`. Bob sends back three `Data` messages with the data for each block, as well as the hashes needed to verify the content in a way similar to the process described above for the metadata feed.


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

IPFS is designed for immutable hash-addressed content, but it provides a mechanism for mutable content using public key addresses (IPNS). IPNS is still under development but some concepts are established. Its premise is much simpler than Hypercore's; rather than encoding the history in a standard form, IPNS simply signs and publishes a content-hash identifier under the public key, therefore creating a `pubkey -> hash` lookup. The referenced content may choose to encode a history, but it is not required and no constraints on branching is enforced. Compared to Hypercore, IPNS may be more user-friendly as it does not suffer from a catastrophic error if the history splits, therefore enabling users to share private keys freely between devices. This comes at the cost that history may be freely rewritten by the dataset author.


# Unresolved questions
[unresolved]: #unresolved-questions

- Is the term "Register" the best for describing the Hypercore structure? What about "Log" or "Feed" ?
- Is there a potential "branch resolution" protocol which could remove the [linear history requirement](#linear-history-requirement) and therefore enable users to share private keys freely between their devices? Explaining the risks of branches to users is difficult.


# Changelog
[changelog]: #changelog

A brief statemnt about current status can go here, follow by a list of dates
when the status line of this DEP changed (in most-recent-last order).

- YYYY-MM-DD: First complete draft submitted for review
