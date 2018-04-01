
Title: **DEP-0000: HTTP Pinning Service API**

Short Name: `0000-http-pinning-service-api`

Type: Standard

Status: Draft (as of 2018-03-31)

Github PR: (add HTTPS link here after PR is opened)

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
    "rel": "user-accounts-api.com/v1",
    "title": "User accounts API",
    "href": "/v1/accounts"
  }, {
    "rel": "datprotocol.com/deps/0000-http-pinning-service-api",
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
POST /register  Register a new account.
POST /verify    Verify the email address of a recently-registered account.
POST /login     Create a new session with an existing account.
POST /logout    End a session.
GET  /account   Get information about the account attached to the current session.
POST /account   Update information about the account attached to the current session.
```

TODO- specify each route in detail.

Full documentation for this API should be made available at https://user-accounts-api.com.

## User registration flow [user-registration-flow]: #user-registration-flow

### Step 1. Register

User POSTS to `/register` with body:

```
{
  email: String
  username: String
  password: String
}
```

Server creates a new account for the user. A random 32-byte email-verification
nonce is created. The user record indicates:

scopes|isEmailVerified|emailVerifyNonce
------|---------------|----------------
none|false|XXX

Server sends an email to the user with the `emailVerifyNonce`. 

Server responds 200 or 204.

### Step 2. Verify (POST /verify)

User POSTS `/v1/verify` with JSON body:

```
{
  username: String, username of the account
  nonce: String, verification nonce
}
```

Server updates user record to indicate:

scopes|isEmailVerified|emailVerifyNonce
------|---------------|----------------
user|true|null

Sever generates a session and session token, and responds 200 with a JSON body:

```
{
  sessionToken: String, users session token
}
```

## Session login flow [session-login-flow]: #session-login-flow

User POSTS to `/login` with body:

```
{
  username: String
  password: String
}
```

Sever generates a session and session token, and responds 200 with a JSON body:

```
{
  sessionToken: String, users session token
}
```


# Dat pinning API
[dat-pinning-api]: #dat-pinning-api

The dat pinning API should provide the following resources:

```
GET  /            List all Dat data pinned by this account.
POST /add         Add a Dat to this account's list of pins.
POST /remove      Remove a Dat from this account's list of pins.
GET  /item/:key   Get information about a Dat in the account's list of pins.
                  Key may be the pubkey or name of the dat.
POST /item/:key   Update information about a Dat in the account's list of pins.
                  Key may be the pubkey or name of the dat.
```

TODO- specify each route in detail.


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


# Changelog
[changelog]: #changelog

- YYYY-MM-DD: First complete draft submitted for review

