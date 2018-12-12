Title: **DEP-0000: Intents**

Short Name: `0000-intents`

Type: Standard

Status: Draft (as of 2018-12-12)

Github PR: [Draft](https://github.com/datprotocol/DEPs/pull/53)

Authors: [Martin Heidegger](https://github.com/martinheidegger)


# Summary
[summary]: #summary

DAT clients can have various purposes when connecting to other peers. A _"backup"_-peer might have the intention to store and
seed as many much of a given DAT as possible, while a _"download-latest-once"_-peer might only try to download the
latest version and leave the network when its done and still a _"browse"_-peer might be only interested in parts of a DAT
at a time a human points at something.

This DEP specifies an additional, optional [Handshake property][Handshake-properties] _(specified in [#8][wireprotocol-pr])_
to announce the intent of the peer.

[Handshake-properties]: https://github.com/pfrazee/DEPs/blob/e761143a051dc9735364d8897baf691bd2ed9d68/proposals/0000-wire-protocol.md#handshake
[wireprotocol-pr]: https://github.com/datprotocol/DEPs/pull/8

# Motivation
[motivation]: #motivation

In user-interfaces, all peers are treated equal. If − _for example_ − a user wants to download a DAT but the only peer is a
_"browse"_-peer then it likely means that the peer will never be able to share all the data as the browsing peer never intends
to provide data to the user. Yet in the UI it is visualised the same way as if a _"backup"_-peer is connected that specifically
intends to server as much of the specified data as possible.

This can also be used for the prioritization of clients and network optimizations. For example: backup seeds might be
prioritized higher 

# Specification
[specification]: #specification

An optional `intent`-property is added to the handshake response that allows to share the peers intent. 

```protobuf
message Handshake {
    // ...
    optional string intent = 6;
}
```

The `intent` is an free string field with pre-specified identifiers:

- `browse`: A browsing client that intends to download and dismiss parts of the downloaded structure at free will.
- `browse-seed`: Similar to `browse` but with the intention to re-seed the downloaded parts.
- `backup`: A client that downloads and re-seeds as many versions of an given hypercore as specified.
- `seed`: A client that downloads and re-seeds only one version _(usually the latest)_ of an hypercore.
- `author`: A client that has the secret keys to author that hypercore.
- `publish`: Similar to `seed` but with the intent also publish outside of the DAT protocol _(i.e. a HTTP service)_
- `backup-publish`: Similar to `backup` but it intends to publish outside of the DAT protocol _(i.e. a HTTP service)_
- `proxy`: A client that connects in order for re-share whatever is requested to further unspecified clients.

For custom strings, the recommendation is to use `non-standard-` as a prefix for intents in order to make sure
that the future amendments to the list will not interfere with non-standard usage.

## Change of an intent
[change-of-intent]: #change-of-intent

In order to change an intent, the client would need to close and re-open a connection.

# Drawbacks
[drawbacks]: #drawbacks

By providing and intents clients can refuse service to selected peers. For example a cliend could decide to
refuse `browse` peers and only accept `browse-seed` peers. Which could lead to browsers to use various
strategies to combat this:

- They could stop identifying properly and only identify as `browse-seed`.
- They could open two connections (one browse ond browse-seed) and only if the experience is similar drop the `browse`
    connection.
- They could blacklist peers that behave like this.


# Changelog
[changelog]: #changelog

- 2018-12-12: Early "WIP" draft circulated on github
- YYYY-MM-DD: First complete draft submitted for review
