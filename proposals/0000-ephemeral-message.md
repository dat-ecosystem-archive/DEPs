
Title: **DEP-0000: Ephemeral Message (Extension Message)**

Short Name: `0000-ephemeral-message-extension`

Type: Informative

Status: Undefined (as of 2018-06-05)

Github PR: https://github.com/datprotocol/DEPs/pull/28

Authors: [Paul Frazee](https://github.com/pfrazee)


# Summary
[summary]: #summary

This DEP defines the non-standard `em` extension message used in the Dat replication protocol. This message provides a way to send arbitrary application data to a peer through an existing connection.


# Motivation
[motivation]: #motivation

While Dat is effective at sharing persistent datasets, applications frequently need to transmit extra information which does not need to persist. This kind of information is known as "ephemeral." Examples include: sending chat messages, proposing changes to a dat, alerting peers to events, broadcasting identity information, and sharing the URLs of related datasets.

This extension message will establish a common mechanism for sending ephemeral messages of an arbitrary composition.


# Reference Documentation
[reference-documentation]: #reference-documentation

This DEP is implemented using the Dat replication protocol's "extension messages." In order to broadcast support for this DEP, a client should declare the `'em'` extension in the replication handshake.

Ephemeral messages can be sent at any time after the connection is established by sending an extension message of type `'em'`. The message may include a payload up to 256 bytes in length. Any additional bytes should be truncated by the receiving client. The payload is a buffer of any encoding.

The client may respond to the message by emitting an event, so that it may be handled by the client's application logic.

No acknowledgment of receipt will be provided (no "ACK").

After publishing this DEP, the "Beaker Browser" will implement a Web API for exposing the `'em'` protocol to applications. It will restrict access so that the application code of a `dat://` site will only be able to send ephemeral messages on connections related to its own content.


# Drawbacks
[drawbacks]: #drawbacks

- This DEP may present privacy concerns, as it may be used to track users in a similar fashion to HTTP Cookies, or be used to exfiltrate data.
- The payload of the `'session-data'` message is not authenticated in any way. The lack of trust must be considered by applications which leverage the data.
- If the recipient of the `'session-data'` message is not authenticated (as is currently the case in all Dat replication connections) the client will not know who is receiving the payload and may broadcast sensitive information.


# Changelog
[changelog]: #changelog

- 2018-06-05: First complete draft submitted for review

