
Title: **DEP-0006: Session Data (Extension Message)**

Short Name: `0006-session-data-extension`

Type: Informative

Status: Draft (as of 2018-06-06)

Github PR: https://github.com/datprotocol/DEPs/pull/27

Authors: [Paul Frazee](https://github.com/pfrazee)


# Summary
[summary]: #summary

This DEP defines the non-standard `session-data` extension message used in the Dat replication protocol. This message provides a way to attach application data to a connection, commonly used for identifying the users and broadcasting personal keys. 


# Motivation
[motivation]: #motivation

Applications frequently need to discover which users of the application are online (presence) in order to establish bidirectional communication. For example, a chat application which uses a shared HyperDB as the channel state may need to broadcast the Hypercore keys of each user in order to authorize the joining chat-users (as in the case of "Cabal"). It would also be useful to broadcast Hyperdrive archive keys (as in the case of "Fritter" and "Rotonde") or even simple plain-text identity (eg "my name is Bob") to be used with other communication mechanisms.

This DEP was motivated by the need for a quick solution to these use-cases. It's expected to be a stepping stone to a more sophisticated solution. It outlines an extension message which broadcasts user and session data. The two primary use-cases considered are:

 1. For small apps to be able to discover peers' identities when shared. Examples include a dat-app containing an event invite, or a collaborative document, which want to know the identities of visitors in order to receive data from them. Scale would be kept small by the fact that the dat-app is only shared with friends. If too many people start showing up, the app could stop authorizing or reading their data. (Identity data broadcasted via this DEP should not be considered highly trust-worthy.)
 2. For dat-apps like Cabal to experiment with authorization policies at a larger scale. Cabal (a chat app) intended to use a HyperDB which any user can "join" as a writer and start posting to. This DEP would make it simple for Cabal to send the local keys of joining users to the owner, to be added to the "channel" HyperDB.


# Reference Documentation
[reference-documentation]: #reference-documentation

This DEP is implemented using the Dat replication protocol's "extension messages." In order to broadcast support for this DEP, a client should declare the `'session-data'` extension in the replication handshake.

Session-data can be announced at any time after the connection is established by sending an extension message of type `'session-data'`. The message may include a payload up to 256 bytes in length. Any additional bytes should be truncated by the receiving client. The payload is a buffer of any encoding. The session-data message should not be sent frequently and a client may choose to rate-limit its handling of the events (this DEP suggests "once per 5 seconds").

The client should maintain a `sessionData` variable on each connection. This variable should be empty when a new connection is established. Any time a `'session-data'` extension message is received, the value of the `sessionData` variable should be updated to contain the payload of the message.

The client may respond to the message by emitting an event, so that it may be handled by the client's application logic. The client should also make the most recent `sessionData` buffer available to the application logic after message is received.

After publishing this DEP, the "Beaker Browser" will implement a Web API for exposing the `'session-data'` protocol to applications. It will restrict access so that the application code of a `dat://` site will only be able to set the session data for connections related to its own content.


# Drawbacks
[drawbacks]: #drawbacks

- This DEP may present privacy concerns, as it may be used to track users in a similar fashion to HTTP Cookies.
- The payload of the `'session-data'` message is not authenticated in any way. If a public key is sent, proof of ownership of the private key is not provided. The lack of trust must be considered by applications which leverage the data.
- If the recipient of the `'session-data'` message is not authenticated (as is currently the case in all Dat replication connections) the client will not know who is receiving the payload and may broadcast sensitive information.


# Rationale and alternatives
[alternatives]: #alternatives

Some applications have used the `peerId` and/or `userData` fields of the replication handshake message in order to broadcast this information. Those mechanisms are unsuitable for Web applications (as in the "Beaker browser") because the sites' applications are not executed reliably prior to the replication handshake. By using an extension message, we provide the same presence & discovery without relying on the timing of the application-code execution.

An alternative approach would be to establish an ephemeral messaging channel, perhaps using a different extension message. This ephemeral channel would broadcast the payload to the client's application code as an event when it is received, but would not retain the most recent payload as session-data. This ephemeral channel would be less effective in Web applications (as in the "Beaker Browser") because it would rely on the application-code being active (loaded in a tab) at time of receipt, whereas the builtin session-data semantic makes it possible for the browser to retain the last payload on the applications' behalf.


# Changelog
[changelog]: #changelog

- 2018-06-06: Merged as draft
- 2018-05-31: First complete draft submitted for review

