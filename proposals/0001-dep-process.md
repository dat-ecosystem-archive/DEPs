
Title: **DEP-0001: The Dat Enhancement Proposal Process**

Short Name: `0001-dep-process`

Type: Process

Status: Draft (as of 2018-01-15)

Github PR: [https://github.com/datprotocol/DEPs/pull/2]()

Authors: TBD

# Summary
[summary]: #summary

The Dat Enhancement Proposal ("DEP") process is how the Dat open source
community comes to (distributed) consensus around technical protocol
enhancements and organizational process.

# Motivation
[motivation]: #motivation

The community around the Dat protocol has grown to the point that standards
documentation and decision making centered around source code (an open
reference implementation) and a single whitepaper is insufficient. A specific
growing pain is the bandwidth of a small number of implementors to respond to
all informal proposals or requests for clarification on their own.

A public DEP process is expected to increase the quality of core technical
protocols and library implementations (by clarifying changes early in the
process and allowing structured review by more individuals), lower the barrier
to additional implementations of the protocols (by clarifying implementation
details and norms not included in the core specification itself), and to make
the development process more transparent, accessible, and scalable to a growing
group of developers and end users.

An additional goal of the process is to empower collaborators who are not core
Dat developers or paid staff to participate in community decision making around
protocols and process. Certain individuals will have special roles and
responsibilities, but should be less of a bottleneck or single-point-of-failure
for the ecosystem as a whole.

# Submitting a Proposal
[submit]: #submit

Before writing and proposing a DEP, which takes some time, it's best to
informally pitch your idea to see if others are already working on something
very similar, or if your idea has been discussed previously. This could take
place over chat, a short github issue, or any other medium.

The process for proposing and debating a new DEP is:

* Fork the [datprotocol/deps](https://github.com/datprotocol/deps) repository
* Copy `0000-template.md` to `proposals/0000-my-proposal.md` (don't chose the
  "next" number, use zero; `my-proposal` should be a stub identifier for the
  proposal)
* Fill in the DEP template. The more details you can fill in at the begining,
  the more feedback reviewers can leave; on the other hand, the sooner you put
  your ideas down in writing, the faster others can point out issues or related
  efforts. Feel free to tweak or expand the structure (headers, content) of the
  document to fit your needs.
* Submit a github pull request for discussion. The initial proposal will likely
  be ammended based on review and comments. Go ahead and `cc:` specific
  community members who you think would be good reviewers, though keep in mind
  everybody's time and attention is finite..
* Build interest and consensus. This part of the process will likely involve
  both discussion on the PR comment thread and elsewhere (IRC, etc).
* Consider drafting or prototyping an implementation to demonstrate your
  proposal and work out all the details. This step isn't strictly necessary,
  however: a proposer does not need to be a developer.
* If the DEP is well-formed and there is sufficient interest (for or against
  the proposal), a team member will assign an DEP number, update the status,
  and merge the PR.  Standards DEPs which need implementation or details to be
  worked out, can be accepted as "Draft"; DEPs with strong acceptance can go
  straight to "Active".
* A "Draft" DEP can be upgraded to "Active" after some time has passed and
  confidence has been increased (eg, unresolved issues have been addressed,
  implementations have been shown in the wild) by opening a PR for discussion
  that sets the new Status.
* Small tweaks (grammar, clarifications) to a merged DEP can take place as
  regular github PRs; revisiting or significantly revising should take place as
  a new DEP. "Draft" and "Process" DEPs have a lower bar for evolution over
  time via direct PR.

All DEPs should have a type ("Standard" or "Process") and a status.

For appropriate DEPs (including *all* Standards DEPs), authors should
explicitly consider and note impacts on:

* Privacy and User Rights: consider reading IETF [RFC 6973] ("Privacy
  Considerations for Internet Protocols") and [RFC 8280] ("Research into Human
  Rights Protocol Considerations")
* Backwards compatibility of on-disk archives and older network clients

[RFC-6973]: https://tools.ietf.org/html/rfc6973
[RFC-8280]: https://tools.ietf.org/html/rfc8280

# Details
[reference-documentation]: #reference-documentation

DEPs should have a type:

* **Standard** for technical changes to the protocol, on-disk formats, or
  public APIs. These are intented to be *proscriptive*, and to clearly
  delineate which features and behaviors are mandatory or optional.
* **Process** for formalizing community processes or other (technical or
  non-technical) decisions. For example, a security vulnerability reporting
  policy, a process for handling conflicts of interest, or procedures for
  mentoring new developers.
* **Informative** for describing conventions, design patterns, existing norms,
  special considerations, etc.

The status of a DEP can be:

* **Pre-Merge**: a well-formed DEP has been written and a PR opened. The
  "Status" line can list "Draft" when in this state.
* **Draft**: PR has been merged and a number assigned, but additional time is
  needed for deeper discussion or more implementation before being fully
  accepted.
* **Active**: adopted or intended for implementation in mainline libraries and
  clients as appropriate
* **Closed**: either consensus was against, a decision was postponed, or the
  authors withdrew their proposal. This could apply to any of: a proposal PR
  that was never merged, a merged Draft (which was never Active), or an Active
  DEP which there is now consensus against without a specific new DEP to
  replace it.
* **Superseded**: a formerly "active" DEP has been made obsolete by a new
  active DEP; the new DEP should specify specific old DEPs that it would
  supersede.

A changelog should be kept in the DEP itself giving the date of any changes of
status.

A template file is provided, but sections can be added or removed as
appropriate for a specific DEP.

The DEP text itself should be permissively licensed; the convention is to use
the Creative Commons Attribution License (CC-BY), with attribution to the major
contributing authors listed.


# Drawbacks
[drawbacks]: #drawbacks

There are already multiple sources of technical documentation: the Dat
[protocol website][proto-website], the Dat [whitepaper][whitepaper], Dat
website [documentation section][docs], the [discussion repo][discussion-repo]
issues, and the [datprotocol github group][datproto-group] (containing, eg, the
`dat.json` repo/spec). Without consensus and consolidation, this would be "yet
another" place to look.

[proto-website]: https://www.datprotocol.com/
[whitepaper]: https://github.com/datproject/docs/blob/master/papers/dat-paper.md
[docs]: https://docs.datproject.org/
[datproto-group]: https://github.com/datprotocol

# Background and References
[references]: #references

The following standards processes were referenced and considered while
designing the DEP process:

* **BitTorrent Enhancement Process** as described in [BEP 1][bep-1].
* **[Rust Language RFC Process][rust-rfc]**
* **[IETF RFC Process][ietf]**
* **[XMPP Standards Process][xmpp]**
* **Python Enhancement Process** documented in [PEP 1][pep-1].

[bep-1]: http://bittorrent.org/beps/bep_0001.html
[rust-rfc]: https://github.com/rust-lang/rfcs
[xmpp]: https://xmpp.org/about/standards-process.html
[ietf]: https://www.ietf.org/about/process-docs.html
[pep-1]: https://www.python.org/dev/peps/pep-0001/

# Unresolved questions
[unresolved]: #unresolved-questions

Who are "core developers"? What is the specific decision making process for
accepting or rejecting a given DEP? Optimistically, it would be clear from
reading a PR discussion thread whether "consensus" has been reached or not, but
this might be ambiguous or intimidating to first-time contributors.

The intention is to retroactively document the entire Dat protocol in the form
of DEPs, but the details and structure for this haven't been worked out.

How mutable should Draft Standards DEPs be over time? What about Process DEPs?
Should there be an additional status ("Living"?) for DEPs that are expected to
evolve, or is this against the whole idea of having specific immutable
documents to reference?

# Changelog
[changelog]: #changelog

- 2018-01-15: TODO: First complete draft submitted for review

