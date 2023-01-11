
Title: **DEP-0000: Dat Nomenclature**

Short Name: `0000-nomenclature`

Type: Informative

Status: Undefined (as of 2018-02-XX)

Github PR: (add HTTPS link here after PR is opened)

Authors: [Joe Hand](https://github.com/joehand)


# Summary
[summary]: #summary

The Dat Protocol is a new protocol, with new concepts and models. To ensure consistency and improve understanding, it is essential that the community agree on the definition of words and how to refer to specific concepts. This DEP aims to begin this process.

# Motivation
[motivation]: #motivation

The Dat community has a variety of new terms and concepts. As these have evolved over time, there is a lack of consistency and definition of terms. This DEP will aim to document existing standard nomenclature, identifying ambiguous terms. Additionally, the DEP will aim to identify and define terms that will be preferred or required in future DEP proposals when discussing certain topics.

By defining Dat nomenclature, we can ensure the writing of the wider Dat community also uses the preferred terms. This will make it easier for new users and contributors to understand the Dat ecosystem. To reduce barriers to entry, this DEP will prefer words that are less technical while conveying the same meaning. 

When addressing terms that have specific technical meaning, such as *hyperdrive*, the DEP will aim to have a definition and identify other related words that should not be used in future DEPs. Terms that have less specific meaning, such as *discovery*, will have a definition and potential synonyms, but leave it to future DEPs to define any terms necessary.

# Usage Documentation
[usage-documentation]: #usage-documentation

### Dat

Dat refers generally to *Dat Project* or *Dat Protocol*. Dat is capitalized, not uppercase, and *Project* or *Protocol* are capitalized. 

Dat can be used by itself if referring generally to a Dat-related entity, such as "the Dat community." When the meaning may be ambiguous, a DEP must include Project or Protocol.

## Other Terms & Concepts


# Rationale
[alternatives]: #alternatives



# Unresolved questions
[unresolved]: #unresolved-questions

- How would it make sense to organize this?
- when discussing the concept of a single "hyperlog register" (aka, an append only log with a specific public key; a Dat archive is made up of two of these), is it called: "a hyperlog"? "a hypercore"? register, log, feed?
- within a dat protocol connection, there can be multiple hyperlog registers synchronized. These each get a channel ID, to disambiguate which messages are about which register. Terms "channel", "feed", "register" are redundant in that context (aka, all refer to the same thing); channel isn't great because it could be referring to a connection at almost any abstraction layer?
- when two hypercore peers connect they can do all sorts of things. do we refer to all of these vaguely as "synchronizing"? "being a member of the swarm"? the later isn't great because it might just be a persistent 1-to-1 connection. Could involve uploading and/or downloading. Maybe just "connected"?
- what are the "entries" within a single hyperlog called, and their number? "version", "entry", "chunk", "element", "node", "id", "index". How to disambiguate between talking about a hyperlog "entry" and a member of a SLEEP file? In the sleep context "entry", "element", and "chunk" also make sense.



# Changelog
[changelog]: #changelog

Early draft status, starting to consolidate nomenclature we need to work out.

- YYYY-MM-DD: First complete draft submitted for review

