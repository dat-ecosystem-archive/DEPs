
Title: **DEP-0000: Hyperdb Timestamps**

Short Name: `0000-hyperdb-timestamps`

Type: Standard

Status: Undefined (as of YYYY-MM-DD)

Github PR: (add HTTPS link here after PR is opened)

Authors: [Bryan Newbold](https://github.com/bnewbold)


# Summary
[summary]: #summary

An optional timestamp field is added to hyperdb "entry" metadata for writers to
self-report the date and time of modifications to the database.


# Motivation
[motivation]: #motivation

The timestamp of changes to Dat archives is not currently captured at any layer
of the technical stack (hyperdrive, hyperdb, or hypercore). Hyperdrive records
the filesystem creation and modification timestamps of individual files, but
these [POSIX][posix]-style fields store pre-existing filesystem-level metadata,
not hyperdrive-level event metadata, and do not track changes like deletion.

By adding an optional field to hyperdb entries, clients and applications will
be able to track the (self-reported) date and time of changes, which are
valuable to human users inspecting the history of a dat archive. Uses of this
metadata might include surfacing recent events to users, calculating activity
metrics over large numbers of archives, and comparing "freshness" between
multiple archives with similar or overlapping content.

This metadata is *not* intended to be trusted for any security or
protocol-layer concerns.

[POSIX]: https://en.wikipedia.org/wiki/POSIX


# Usage Documentation
[usage-documentation]: #usage-documentation

Implementation libraries expose timestamp information at higher protocol
levels. The Hyperdb library also exposes a method to fetch the timestamp for a
given revision/sequence of the log, and options to control the inclusion of
timestamps when creating databases or making modifications.

It is important to note that timestamps are self-generated and reported by the
writer's computer, which may be misconfigured or have significant local clock
skew. This means that timestamps can end up inaccurate, out of order, or
entirely bogus (eg, in the case of a device with no datetime configured at all,
which might default to the year 1970).


# Reference Documentation
[reference-documentation]: #reference-documentation

The timestamp would be an additional `optional` field to the `Entry` and
`InflatedEntry` protobuf fields in the hyperdb protocol, with type `uint64`.
The number represents the number of milliseconds since the UNIX epoch, in the
UTC timezone. The timestamp would be calculated at the time the protobuf
message is created for appending to the hypercore log.


# Drawbacks
[drawbacks]: #drawbacks

The marginal storage and bandwidth overhead to implement timestamps would be
very small for Dat and hyperdrive applications, but could be non-trivial for
some hyperdb key/value store applications with very small values. Even in this
case, the overhead would be on the order of 5 bytes per key.

See [Privacy and Security Concerns][privacy].


# Privacy and Security Concerns
[privacy]: #privacy

Timestamps can be used as a side-band to deanonymize users, by leaking
information about their local timezone and computer usage habits.
High-resolution timestamps can be correlated with network traffic or real-life
events to identify nodes.

Users and applications could "fuzz" (add random error to), truncate, or
entirely disable timestamping to mitigate these issues, but these would depend
on user and developer awareness of the issue.


# Rationale and alternatives
[alternatives]: #alternatives

The usefulness of "self-reported" (as opposed to network-verified) timestamps
is demonstrated by the git version control software.

The timestamp could have additional resolution (eg, nanoseconds) at the cost of
additional storage. Seconds seems like a good design trade-off between
efficiency and value to human users. Timestamps could also be stored in
floating point, but this removes the potential efficiency of `varint` encoding.

The timestamp field is added at the hyperdb layer instead of hyperdrive (in the
`Stat` metadata) because in the hyperdb version of hyperdrive, a deletion is
represented as a non-existant `value` (aka, no  `Stat` at all). If timestamps
aren't stored at the hyperdb layer, then deletions can't be timestamped.

The consistent use of UTC time requires implementations to convert to local
time, but avoids the complexities of local timezone representations. The
behavior around leap seconds is intentionally not specified here, to allow use
of "clock smearing" or operating system default transitions to TAI or other
leap-second-free time standards.


# Unresolved questions
[unresolved]: #unresolved-questions

Should this instead live at the hypercore layer? Protocol designers have so far
been hesitant to add additional features or complexity to that protocol layer.

Should a new "wrapper" message type be added to hyperdrive to allow
timestamping of deletions as well as insertions. This could look something
like:

```protobuf
message DriveEvent {
    message Stat {
        required uint32 mode = 1;
        optional uint32 uid = 2;
        optional uint32 gid = 3;
        optional uint64 size = 4;
        optional uint64 blocks = 5;
        optional uint64 offset = 6;
        optional uint64 byteOffset = 7;
        optional uint64 mtime = 8;
        optional uint64 ctime = 9;
    }

    optional Stat stat = 1;
    optional uint64 timestamp = 2;
}
```

Is there a better practice today (in 2018) for dealing with leap seconds in
specifications?


# Changelog
[changelog]: #changelog

- 2018-03-17: First submitted for comment.

