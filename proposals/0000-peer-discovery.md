
Title: **DEP-0000: Peer Discovery**

Short Name: `0000-peer-discovery`

Type: Informative

Status: Undefined (as of 2018-02-06)

Github PR: (add HTTPS link here after PR is opened)

Authors: [Paul Frazee](https://github.com/pfrazee)


# Summary
[summary]: #summary

An important aspect of Dat's networking is peer discovery, the techniques that peers use to find each other. Peer discovery means finding the IP and port of data sources online that have a copy of that data you are looking for. You can then connect to them and begin exchanging data. By using peer discovery techniques Dat is able to create a network where data can be discovered even if the original data source disappears.

Peer discovery can happen over many kinds of networks. In the Dat implementation we currently implement discovery on top of three networks:

- Multicast DNS - Useful for discovering peers on local networks
- DNS name servers - An Internet standard mechanism for resolving keys to addresses
- Kademlia Mainline Distributed Hash Table - Less central points of failure, increases probability of Dat working even if DNS servers are unreachable

Additional discovery networks can be implemented as needed. We chose the above three as a starting point to have a complementary mix of strategies to increase the probability of source discovery.


# Peer discovery methods
[peer-discovery-methods]: #peer-discovery-methods

Dat uses multiple discovery networks, to provide redundancy and to suit differing network needs. There is no restriction on which discovery solutions are allowed, but at time of writing there are three in active use.


## Multicast DNS
[multicast-dns]: #multicast-dns

Multicast DNS (mDNS) resolves host names to IP addresses within small networks without a local name server. It is a zero-configuration service, using essentially the same interfaces, packet formats and operating semantics as unicast DNS. The mDNS protocol is published as [RFC 6762](https://tools.ietf.org/html/rfc6762) and is built on multicast UDP.

Dat treats Hypercore public keys as domain names on the mDNS protocol. Therefore, peer discovery is an IP lookup for a given public key name. Currently the public key is encoded to hex and truncated to 40 bytes. The domain name format used is:

```
{PUBKEY}.dat.local
```

Dat uses the `TXT` record type. A query is submitted as a simple `TXT` query for `{PUBKEY}.dat.local`. The response provides a peer-listing which will only include the local node, if it is actively hosting the requested Hypercore.


### TXT data encoding
[dns-txt-data-encoding]: #dns-txt-data-encoding

TXT record data is encoded as key/values using [RFC 6763](https://tools.ietf.org/html/rfc6763#section-6) DNS-SD encoding.

Peer listings are a base64-encoded buffer of 6-byte peer items. Each peer item is packed as follows:

```
{4 bytes: IPv4 address}{2 bytes: port}
```


## DNS name servers
[dns-name-servers]: #dns-name-servers

While mDNS is effective for tracking and discovering peers on the Local Area Network, it does not work for the global Internet. For that, Dat's solution is to use DNS name servers with custom behaviors. These servers are maintained by the Dat protocol Working Group members, but may be reconfigured to use other servers.

The DNS protocol queries serve to lookup peers, announce swarm membership, and subscribe to push-updates. To interact with a DNS name server, a client must first "probe" the server for a session token. This is described in the "Session token exchange" section.

Much of the details of the DNS discovery is shared with mDNS, including the TXT data encoding (see above). At time of writing, the DNS and mDNS discovery tools are implemented in one codebase in the active Dat implementation.


### Session token exchange
[dns-name-server-session-token-exchange]: #dns-name-server-session-token-exchange

DNS is built on UDP, a sessionless connection protocol. Because the DNS peer discovery protocol involves registration for future messages, it's important that the DNS server verifies the IP of a registrar. Otherwise, a malicious peer could spoof its IP in order to register other devices for receiving messages, leading to potential DoS attacks.

To verify the addresses of clients, the DNS discovery protocol uses a session token exchange. All clients must first request a token before sending protocol messages. The server will generate the token using the following algorithm:

```
sha256(secret + client-address)
```

The secret should be generated from random.

This token must be included in queries which include mutation fields in the "additional" section. (Simple lookups do not require the token.) By requiring the token, we prove that the sender's IP is not spoofed, as it *must* provide a valid address in order to receive the token during the session token exchange.

The token is requested by sending a `TXT` record to the DNS server with a target name of `"dat.local"`. The server will respond with the token, plus the port and address of the sending device (which are useful as a "whoami").

Over time, the server will rotate the secret it uses to generate tokens. In order to update clients' tokens, every response includes the latest token. The client should update its token with every response it receives. (It's advised that the server keeps the most recently expired secre so that old tokens can be accepted and replaced smoothly.)


### Lookup query
[dns-name-server-lookup-query]: #dns-name-server-lookup-query

To request the current list of known peers for a pubkey, send a `TXT` question query with `{PUBKEY}.dat.local` as the name. Currently the public key is encoded to hex and truncated to 40 bytes. You will receive a response that includes a full peer listing and the latest token. See "TXT data encoding" above for information about encoding.

Every query may include a `TXT` "additional" section which includes the session token and any behavior fields (described below).


#### Subscribe flag
[dns-name-server-subscribe-flag]: #dns-name-server-subscribe-flag

The `subscribe` flag instructs the DNS name server to add the device to the list of active listeners for the given Hypercore. Any time a new peer is announced, the server will "push" a notification to the device.

The push is sent as the "additional" section of an `SRV` query. It contains as its data the `target` (address) and `port` of the new peer.

If a `TXT` lookup query is sent with an "additional" section that does not have the `subscribe` flag, that is treated as an "unsubscribe" message and the device is removed from the active listeners.

TODO- what's the TTL?


#### Announce field
[dns-name-server-announce-field]: #dns-name-server-announce-field

The `announce` field instructs the DNS name server to add the device to the list of active hosting peers for the given Hypercore. Its value should be the port from which the device is listening. Multiple ports may be announced using separate queries. Upon announce, the new peer is pushed to any subscribed devices using an `SRV` query.

TODO- how long till announce records expire? Should the client reannounce periodically?


#### Unannounce field
[dns-name-server-unannounce-field]: #dns-name-server-unannounce-field

The `unannounce` field instructs the DNS name server to remove the device from the list of active hosting peers. Its value should be the port from which the device was previously listening.


## Kademlia Mainline DHT
[kademlia-mainline-dht]: #kademlia-mainline-dht

Mainline DHT is the name given to the Kademlia-based Distributed Hash Table (DHT) used by BitTorrent clients to find peers. Dat has adopted it temporarily to track peers in its own network. You can find the specification at [BEP 0005](http://www.bittorrent.org/beps/bep_0005.html).

There are some issues with Dat's use of Mainline which limit the usefulness of its function. BitTorrent uses a 20 byte sha1 hash to identify torrents, while Dat uses a 32 byte public key to identify Hypercore registers. As a result, Dat has to truncate its keys to the first 20 bytes, leading to false positives when connecting to peers.


# Privacy concerns
[privacy-concerns]: #privacy-concerns

Peer discovery networks reveal the participants in a Dat swarm to any device which can access the network. This presents a privacy risk for users who may not want to have their activity broadcasted.

There are many solutions to explore to this issue:

 - Private discovery networks. This will reduce the number of possible data sources, which reduces the success rate of discovery, but also limits the exposure of the user's activity.
 - Proxy services. This will increase the latency of traffic and will expose all activity to the proxy, but it will mask the user's activity among the activity of all proxy users.


# Unresolved questions
[unresolved]: #unresolved-questions

 - Does the DNS network *need* to truncate the public key to 40 bytes? Could we fit the full 64 bytes by using another level of subdomain?


# Changelog
[changelog]: #changelog

A brief statemnt about current status can go here, follow by a list of dates
when the status line of this DEP changed (in most-recent-last order).

- YYYY-MM-DD: First complete draft submitted for review
