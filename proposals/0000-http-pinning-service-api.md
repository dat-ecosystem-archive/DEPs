
Title: **DEP-0000: HTTP Pinning Service API**

Short Name: `0000-http-pinning-service-api`

Type: Standard

Status: Draft (as of 2018-03-31)

Github PR: https://github.com/datprotocol/DEPs/pull/19

Authors: Paul Frazee


# Summary
[summary]: #summary

An HTTP API for adding and removing Dat data.


# Motivation
[motivation]: #motivation

Users frequently make use of "pinning services" to keep their Dat data online
independently of their personal devices. By specifying a standard API for
accessing pinning services, we can integrate interfaces for these services to
Dat clients (including the Dat CLI and Beaker Browser). For example, in the Dat
CLI, it will be possible to execute commands such as:

```
dat publish --name myarchive my-pinning-service.com
```


# Service description (PSA) document
[service-description]: #service-description

Servers should host the PSA service-description document at `/.well-known/psa`.
It may be fetched using a GET request. This document will fit the following schema:

```json
{
  "PSA": 1,
  "title": "My Pinning Service",
  "description": "Keep your Dats online!",
  "links": [{
    "rel": "https://archive.org/services/purl/purl/datprotocol/spec/pinning-service-account-api",
    "title": "User accounts API",
    "href": "/v1/accounts"
  }, {
    "rel": "https://archive.org/services/purl/purl/datprotocol/spec/pinning-service-dats-api",
    "title": "Dat pinning API",
    "href": "/v1/dats"
  }]
}
```

You can read more about the [PSA Service Discovery
Protocol](https://github.com/beakerbrowser/beaker/wiki/PSA-Web-Service-Discovery-Protocol).

The PSA document must provide links to two API resources: the User Accounts
API, and the Dat Pinning API. These resources should be indicated by the
`user-accounts-api.com/v1` and `datprotocol.com/deps/0000-http-pinning-service-api`
rel-types, respectively. (These rel-types will need to be updated
with the final URLs for their specifications.) If either API is absent from
the PSA document, the service will be rejected.


# User accounts API
[user-accounts-api]: #user-accounts-api

The user-accounts API should provide the following resources:

```
POST /login     Create a new session with an existing account.
POST /logout    End a session.
GET  /account   Get information about the account attached to the current session.
```

## POST /login
[user-accounts-api-post-login]: #user-accounts-api-post-login

Create a new session with an existing account.

Request body (JSON). All fields required:

```
{
  username: String
  password: String
}
```

Handler should generate a session and return the identifier in the response.
Response body (JSON):

```
{
  sessionToken: String
}
```

## POST /logout
[user-accounts-api-post-logout]: #user-accounts-api-post-logout

End a session.

Request should include [authentication header](#authentication).

## GET /account
[user-accounts-api-get-account]: #user-accounts-api-get-account

Get information about the account attached to the current session.

Request should include [authentication header](#authentication).

Response body (JSON):

```
{
  email: String, the accounts email (required)
  username: String, the accounts username (required)
  diskUsage: Number, how much disk space has the user's data taken? (optional)
  diskQuota: Number, how much disk space can the user's data take? (optional)
  updatedAt: Number, the Unix timestamp of the last time the user account was updated (optional)
  createdAt: Number, the Unix timestamp of the last time the user account was updated (optional)
}
```

If `diskQuota` is not included or is set to 0, the service is acting as a "registry" and will not host the files.


# Dat pinning API
[dat-pinning-api]: #dat-pinning-api

The dat pinning API should provide the following resources:

```
GET  /            List all Dat data pinned by this account.
POST /add         Add a Dat to this account's list of pins.
POST /remove      Remove a Dat from this account's list of pins.
GET  /item/:key   Get information about a Dat in the account's list of pins.
POST /item/:key   Update information about a Dat in the account's list of pins.
```

## GET /
[dat-pinning-api-get-root]: #dat-pinning-api-get-root

List all Dat data pinned by this account.

Request should include [authentication header](#authentication).

Response body (JSON):

```
{
  items: [{
    url: String, dat url
    name: String, optional shortname assigned by the user
    title: String, optional title extracted from the dat's manifest file
    description: String, optional description extracted from the dat's manifest file
    additionalUrls: Array of Strings, optional list of URLs the dat can be accessed at
  }]
}
```

## POST /add
[dat-pinning-api-post-add]: #dat-pinning-api-post-add

Add a Dat to this account's list of pins.

Request should include [authentication header](#authentication).
Request body (JSON):

```
{
  url: String, required url/key of the dat
  name: String, optional shortname for the archive
  domains: Array of Strings, optional list of domain-names the dat should be made available at
}
```

## POST /remove
[dat-pinning-api-post-remove]: #dat-pinning-api-post-remove

Remove a Dat from this account's list of pins.

Request should include [authentication header](#authentication).
Request body (JSON):

```
{
  url: String, required url/key of the dat
}
```

## GET /item/:key
[dat-pinning-api-get-item]: #dat-pinning-api-get-item

Get information about a Dat in the account's list of pins. Key must be the
pubkey of the dat.

Response body (JSON):

```
{
  url: String, dat url
  name: String, optional shortname assigned by the user
  title: String, optional title extracted from the dat's manifest file
  description: String, optional description extracted from the dat's manifest file
  additionalUrls: Array of Strings, optional list of URLs the dat can be accessed at
}
```

## POST /item/:key
[dat-pinning-api-post-item]: #dat-pinning-api-post-item

Update information about a Dat in the account's list of pins. Key must be the
pubkey of the dat.

Request body (JSON):

```
{
  name: String, optional shortname for the archive
  domains: Array of Strings, optional list of domain-names the dat should be made available at
}
```


# Authentication
[authentication]: #authentication

Clients should use the [User accounts API](#user-accounts-api) to fetch a
session token from the service. This token should be included in the
`Authentication` header using the `Bearer` scheme.


# Error responses
[error-responses]: #error-responses

All error responses should respond with a JSON body which matches the
following schema:

```
{
  message: String
}
```

The content of `message` will be displayed to the user. It should explain the
error and, if appropriate, give steps for solving the issue. Other fields may
be included in the response.


# Rationale and alternatives
[alternatives]: #alternatives

- The motivations and drawbacks of the PSA Service Document are discussed
[here](https://github.com/beakerbrowser/beaker/wiki/PSA-Web-Service-Discovery-Protocol#motivation).
Without a description format, it becomes difficult to handle user
authentication. We would probably need to use the HTTP Basic scheme and remove
any mechanisms for registering new accounts.


# Unresolved questions
[unresolved]: #unresolved-questions

- Does the registration flow need to be included in the spec?


# Changelog
[changelog]: #changelog

- YYYY-MM-DD: First complete draft submitted for review

