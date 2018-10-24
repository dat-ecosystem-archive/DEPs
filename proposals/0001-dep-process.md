
Title: **DEP-0001: The Dat Enhancement Proposal Process**

Short Name: `0001-dep-process`

Type: Process

Status: Draft (as of 2018-01-30)

Github PR: [Draft](https://github.com/datprotocol/DEPs/pull/2)

Authors: [Dat Protocol Working Group][wg]:
[Tara Vancil](https://github.com/taravancil),
[Paul Frazee](https://github.com/pfrazee),
[Mathias Buus](https://github.com/mafintosh),
[Karissa McKelvey](https://github.com/karissa),
[Joe Hand](https://github.com/joehand),
[Danielle Robinson](https://github.com/daniellecrobinson),
[Bryan Newbold](https://github.com/bnewbold)

[wg]: https://github.com/datprotocol/working-group


# Summary
[summary]: #summary

The Dat Enhancement Proposal ("DEP") process is how the Dat open source
community comes to (distributed) consensus around technical protocol
enhancements and organizational process.


# Motivation
[motivation]: #motivation

The Dat protocol is still a living standard. A transparent process is needed
for community members to understand what changes are in the pipeline and how
new ideas might come to fruition.

The core protocol is being used and extended by several projects with differing
priorities and use cases. At the same time, lead developer time is very scarce.
There is a need to parallelize design and implementation work between projects,
which requires better coordination (process) and communication of technical
details (standards). There is also an increasing need to be legible to and
accessible to parties outside the existing Dat ecosystem.

With growing use, the logistics of rolling out protocol changes and
backwards-incompatible changes becomes more difficult, but at the same time
more important to coordinate smoothly. Planning requires clear communication of
change ahead of time.

A public DEP process is expected to improve coordination and planning by
setting clear expectations for documentation of protocol changes and
extensions. The technical quality of the protocol itself should be improved by
increasing the number of people who can view and understand proposals at each
step of the process. The barrier to entry for independent implementations
should be lower, allowing new technical and user communities to adopt the
protocol. The overall development and decision making process should be more
transparent, accessible, and scalable to a growing group of application
developers and end users


# How To Submit a Proposal
[submit]: #submit

As a first step, before drafting a DEP or implementing experimental new
protocol features, it's helpful to informally pitch your idea to see if others
in the community are already thinking something similar, or have discussed the
same idea in the past. This discussion could happen over chat, Github issues,
blog posts, or other channels. If you can recruit collaborators and work out
some of the details, all the better. This period could be called **pre-DEP**.

Once your idea has been flushed out, the process for proposing and debating a
new DEP is:

1. Use git to fork the [datprotocol/deps](https://github.com/datprotocol/deps)
   repository
1. Copy `0000-template.md` to `proposals/0000-my-proposal.md` (don't choose the
   "next" number, use zero; `my-proposal` should be a stub identifier for the
   proposal.)
1. Fill in the DEP template. All proposals should have a Type and Status (see
   below for details). Feel free to tweak or expand the structure (headers,
   content) of the document to fit your needs, but your proposal should be
   "complete" before submission.
1. You can create an informal WIP (work in progress) PR (pull-request)
   whenever you like for early feedback and discussion, but there is no
   expectation that your proposal will be given detailed review until it is
   complete.
1. When you are ready, submit your complete proposal for review (this could be
   opening a PR or removing WIP status from an existing one), with the intent
   of being accepted with "Draft" status.
   An editor (somebody who is an owner of the DEPs repository) will look over
   your proposal for completeness; if acceptable, they will assign one or more
   reviewers.
   At this stage, two outcomes are the most likely: your proposal
   is merged with "Draft" status, or declined.  This first stage of the review
   is expected to take **3 weeks at most** from when reviewers were assigned.
   It is appropriate to propose specific community members to review your
   proposal. The submitter can withdraw a proposal at any time. If accepted, a
   DEP number will be assigned and the PR merged. If declined, reviewers may
   give feedback and/or invite proposers to "significantly revise and resubmit".
1. While in draft status, proposals can be experimented with. Small corrections
   and clarifications can be submitted by PR expect to be merged quickly if
   they are reasonable and don't change the broad behavior or semantics of the
   proposal; larger changes should be re-submitted as Superseding proposals.
1. When it seems appropriate, a PR can be submitted to upgrade the status of a
   "Draft" to "Active". At this time a final review will take place, with the
   outcome being that a proposal stays "Draft" or becomes "Active".
   This review period is shorter (**2 weeks at most**), as everybody is
   expected to be more familiar with the proposal at this point.
   Small changes to the DEP can be included, but it's expected that
   this is in broad strokes the same proposal that was reviewed earlier (if
   not, a new "Draft" should be proposed that Supersedes).
1. Small tweaks (grammar, clarifications) to a merged DEP can take place as
   regular Github PRs; revisiting or significantly revising should take place
   as a new DEP. Draft, Process, and Informational DEPs have a lower bar for
   evolution over time via direct PR.

# Details
[reference-documentation]: #reference-documentation

## Types and Statuses
[types]: #types

DEPs should have a type:

* **Standard** for technical changes to the protocol, on-disk formats, or
  public APIs. These are intended to be *prescriptive*, and to clearly
  delineate which features and behaviors are mandatory or optional.
* **Process** for formalizing community processes or other (technical or
  non-technical) decisions. For example, a security vulnerability reporting
  policy, a process for handling conflicts of interest, or procedures for
  mentoring new developers.
* **Informative** for describing conventions, design patterns, existing norms,
  special considerations, etc.

The status of a DEP can be:

* **Pre-Merge**: a well-formed DEP has been written and a PR opened. The
  "Status" line can list Draft when in this state.
* **Draft**: PR has been merged and a number assigned, but additional time is
  needed for deeper discussion or more implementation before being "normative"
  and expected for implementation. It is acceptable to have "competing"
  proposals in this state at the same time.
* **Active**: adopted or intended for implementation in mainline libraries and
  clients as appropriate. Again, DEPs should clarify which aspects of
  themselves are optional or required for well-behaved clients.
* **Closed**: either consensus was against, a decision was postponed, or the
  authors withdrew their proposal. This could apply to any of: a proposal PR
  that was never merged, a merged Draft (which was never Active), or an Active
  DEP which there is now consensus against without a specific new DEP to
  replace it.
* **Superseded**: a formerly Active DEP has been made obsolete by a new
  Active DEP; the new DEP should specify specific old DEPs that it would
  supersede.

## Content and Structure
[content]: #content

A changelog should be kept in the DEP itself giving the date of any changes of
status.

A template file is provided, but sections can be added or removed as
appropriate for a specific DEP.

The DEP text itself should be permissively licensed; the convention is to use
the Creative Commons Attribution License (CC-BY), with attribution to the major
contributing authors listed.

For appropriate DEPs (including *all* Standards DEPs), authors should
explicitly consider and note impacts on:

* Privacy and User Rights. Consider reading IETF [RFC 6973] ("Privacy
  Considerations for Internet Protocols") and [RFC 8280] ("Research into Human
  Rights Protocol Considerations".)
* Backwards compatibility of on-disk archives and older network clients. If a
  backwards-incompatible change is proposed, a migration plan should be
  sketched out in the proposal.
* Security.

[RFC-6973]: https://tools.ietf.org/html/rfc6973
[RFC-8280]: https://tools.ietf.org/html/rfc8280


## Acceptance Criteria
[criteria]: #criteria

The criteria for a proposal being accepted as a Draft are, at a minimum, that
the proposal is complete, understandable, unambiguous, and relevant. There is a
good faith assumption that the submitter believes that the proposal could
actually be adopted and put to beneficial use. An editor may screen proposals
before passing on to the group for review.

For Standards and Process DEPs, Draft proposals should be specific enough that
they can be prototyped and experimented with (e.g. in a pilot program or test
network), but it isn't expected that all details have been worked out. Working
code is helpful but not required.

For a Draft to migrate to Active, there is an expectation that the proposal has
been demonstrated (e.g. in working code, though not necessarily in "official"
libraries yet), that the proposal will be the "new normal" and expected
behavior going forward, and that the chance of unforeseen issues arising during
complete adoption is low.


## Decision Making Process
[power]: #power

There exists a [Protocol Working Group (WG)][wg] (WG) which makes DEP status
decisions; see the Github repository for a list of current members and the
governance process.

By no means should working group members be the only people reviewing or giving
feedback on proposals.

When deciding on Draft status, at least one WG member must review the entire
proposal in detail, give feedback, and give informed approval. If no detailed
review takes place in the fixed time window, the default is to close (reject)
until a member is willing to commit to review. Any WG member can request
revisions or clarifications (blocking acceptance until addressed), and any
member can block. In other words, consensus requires that at least one member
actively approve, and any member can block, but it isn't required to have every
member review and actively given an opinion. This is referred to as the "loose
consensus" model.

For Active status, the expectation is that all working group members will
review the proposal and actively participate in consensus. In the event that
not all members can participate, the default is again negative. Any member can
block. This is referred to as the "complete consensus" model.

For all other status changes, at least one WG member must vouch for or approve
the change ("loose consensus"). If there is unambiguous consensus (or, eg, a
DEP is documenting already adopted practice), a DEP can move directly to Active
status (following the "complete consensus" process).

In all cases, if there is a deadlock (a block can not be overcome after
further discussion), there is an option to override the block by a vote. The
details of this process are left to description in the Working Group
repository, and are expected to be used only in exceptional cases (eg, are not
invoked by default if a member blocks).

Proposals are expected to be open for at least three days (72 hours) for
comment (and longer to accommodate special circumstances, like holidays). Vetoes
(blocks) can happen up to a week after initially being submitted for review,
which might be retroactive if the proposal was accepted (by other WG members)
very quickly.


# Rationale
[rationale]: #rationale

In the design space of RFC processes, there are two decision points determining
the formality and maturity of accepted standards. The first is, does draft
status mean *could* be implemented, or *has* been implemented? We chose
"could". Secondly, for active (or "final") status, is the proposal *expected*
to be dominant in the wild, or *is it already* dominant in the wild? We chose
"expected". In both cases we are emphasizing clarification and stabilization of
new ideas, as opposed to enforcing interoperability of competing formulations
of the same idea. In short, we expect DEPs to lead (rather than tail)
implementation.

Time limits and default outcomes are used to prevent proposals could get stuck
in an ambiguous indefinite state anywhere along the process. "Draft" status is
considered a stable state to linger in.

Setting expectations for "completeness" of proposals, having an editor quickly
skim proposals before jumping in to a full review, and acknowledging an explicit
"revise and re-submit" workflow are all attempts to head off the situation of
partial proposals being submitted and then significantly revised, which places
extra (time) burden on reviewers.


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
[discussion-repo]: https://github.com/datproject/discussions/issues


# Background and References
[references]: #references

The following standards processes were referenced and considered while
designing the DEP process:

* **BitTorrent Enhancement Process** as described in [BEP 1][bep-1]. The
  Bittorrent protocol has a lot of technical similarities to Dat, and as a
  single protocol family (not a language or full-stack system) is one of the
  most similar in scope. The de facto BEP model is that Drafts are very stable
  and widely adopted; only the most universal core components are Final. The
  DEP process bases it's type categories on the BEP process. There is a single
  editor and an explicit BDFL in the BEP process.
* The **[Rust Language RFC Process][rust-rfc]** is relatively new, but has had
  a huge volume of proposals, rivaling even the IETF. The process is relatively
  lightweight and happens entirely on Github; it is the most similar to the DEP
  process proposed here, in terms of Draft/Active distinction. Rust has strong
  organizational backing with defined leadership roles; proposals are reviewed
  by specific sub-teams.
* **[IETF RFC Process][ietf]**: perhaps the oldest and best known RFC process,
  under the motto of "rough consensus and working code". The process is very
  bespoke (involving custom file formats and software) and heavy on process
  (with working groups and in-person meetings).
* **[XMPP Standards Process][xmpp]**: has the interesting sub-pattern of
  regularly updated (annual) standards. XMPP is also a protocol, like
  Bittorrent. The protocol was designed for easy extension, and at various
  points has seen adoption, extension, and pressure from powerful entities.
* **Python Enhancement Process** documented in [PEP 1][pep-1]. PEPs are
  relatively broad in scope (they often speak to process and organizational
  dynamics), and are widely cited directly by name. Proposals are usually
  debated in great detail on mailing lists before being proposed. Python has a
  BDFL (benevolent dictator for life) who has final say over proposals, though
  he sometimes delegates to deputies.
* The **W3C** is a paid membership organization which, like the IETF, is made
  up of entities large and small, for-profit and altruistic, with decent
  regional diversity. W3C standards are often rather large and verbose
  documents, and tend to tail (rather than lead) implementation.


[bep-1]: http://bittorrent.org/beps/bep_0001.html
[rust-rfc]: https://github.com/rust-lang/rfcs
[xmpp]: https://xmpp.org/about/standards-process.html
[ietf]: https://www.ietf.org/about/process-docs.html
[pep-1]: https://www.python.org/dev/peps/pep-0001/


# Unresolved questions
[unresolved]: #unresolved-questions

How mutable should "Draft" Standards DEPs be over time? What about
Informational DEPs? Should there be an additional status ("Living"?) for DEPs
that are expected to evolve, or is this against the whole philosophy of having
specific stable documents to reference? This is expected to be clarified while
this DEP itself is in Draft status.

Does "Active" status mean that implementation is *mandatory*, and that features
*must* be implemented unless they are explicitly optional? How would this
expectation be enforced for third-party software? This is expected to be
clarified when concrete examples arise.

# Changelog
[changelog]: #changelog

- 2018-01-24: DEP process and governance model is discussed by Working Group.
  First full draft of DEP-0001 submitted for review.

