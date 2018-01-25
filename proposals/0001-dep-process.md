
Title: **DEP-0001: The Dat Enhancement Proposal Process**

Short Name: `0001-dep-process`

Type: Process

Status: Draft (as of 2018-01-15)

Github PR: [Draft](https://github.com/datprotocol/DEPs/pull/2)

Authors: [Dat Protocol Working Group][wg]:
[Tara Vancil](https://github.com/taravancil),
[Paul Frazee](https://github.com/pfrazee),
[Mathias Buus](https://github.com/mafintosh),
[Karissa McKelvey](https://github.com/karissa),
[Joe Hand](https://github.com/karissa),
[Danielle Robinson](#),
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

A public DEP process is expected to improve coordination and planning by
setting clear expectations for documentation of protocol changes and
extensions. The technical quality of the protocol itself should be improved by
increasing the number of people who can view and understand proposals at each
step of the process. The barrier to entry for independent implementations
should be lower, allowing new technical and user communities to adopt the
protocol. The overall developent and decision making process should be more
transparent, accessible, and scalable to a growing group of application
developers and end users

# Submitting a Proposal
[submit]: #submit

As a first step, before drafting a DEP or implementing experimental new
protocol features, it's helpful to informally pitch your idea to see if others
in the community are already thinking something similar, or have discussed the
same idea in the past. This discussion could happen over chat, github issues,
blog posts, or other channels. If you can recruit collaborators and work out
some of the details, all the better. This period could be called **pre-DEP**.

Once your idea has been flushed out, the process for proposing and debating a
new DEP is:

1. Use git to fork the [datprotocol/deps](https://github.com/datprotocol/deps)
   repository
1. Copy `0000-template.md` to `proposals/0000-my-proposal.md` (don't chose the
   "next" number, use zero; `my-proposal` should be a stub identifier for the
   proposal)
1. Fill in the DEP template. All proposals should have a Type and Status (see
   below for details). Feel free to tweak or expand the structure (headers,
   content) of the document to fit your needs, but your proposal should be
   "complete" before submission.
1. You can submit an informal WIP (work in progress) PR whenever you like for
   early feedback and discussion, but there is no expectation that your
   proposal will be given detailed review until it is complete.
1. When you are ready, submit your complete proposal for review (this could be
   opening a PR or removing WIP status from an existing one). An editor will
   look over your proposal for completeness; if acceptable, they will assign
   one or more reviewers. At this stage, there are two primary outcomes: your
   proposal is merged with "Draft", or declined. Submited proposals are
   expected to be complete, understandable, and relevent; see below for more
   details. This early stage of the review is expected to take **3 weeks at
   most** from when reviewers were assigned. It is appropriate to propose
   specific community members to review your proposal. The submiter can
   withdraw a proposal at any time. If accepted, a DEP number will be assigned
   and the PR merged. If there is unambiguous consensus (or, eg, a DEP is
   documenting already adopted practice), a DEP can move directly to Active at
   this stage.
1. While in draft status, proposals can be experimented with. Small corrections
   and clarifications can be submitted by PR expect to be merged quickly if
   they are reasonable and don't change the broad behavior or semantics of the
   proposal; larger changes should be re-submitted as Superceding proposals.
1. When it seems approrpriate, a PR can be submitted to upgrade the status of a
   Draft to Active. At this time a final review will take place, with the
   outcome being that a prosal stays a Draft or is Active. It's also possible
   for a Draft to be Closed (usuall by a specific PR to propose this). This
   review period is shorter (**2 weeks maximum**), as the relevant reviewers
   are expected to be familiar with the proposal at this point. Reasonably
   sized changes to the DEP can be included, but it's expected that this is
   in broad strokes the same proposal that was reviewed earlier (if not, a new
   Draft should be proposed that Supercedes).
1. Small tweaks (grammar, clarifications) to a merged DEP can take place as
   regular github PRs; revisiting or significantly revising should take place
   as a new DEP. Draft, Process, and Informational DEPs have a lower bar for
   evolution over time via direct PR.

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
  adopted.
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


# Adoption Criteria
[criteria]: #criteria

The criteria for a proposal being accepted as a Draft are, at a minimum, that
the proposal is complete, understandable, unambiguous, and relevant. There is a
good faith assumption that the submiter believes that the proposal could
actually be adopted and put to beneficial use. An editor (any member of the
Protocol Working Group) screens proposals before to the group for review.

For Standards and Process DEPs, Draft proposals should be specific enough that
it could be prototyped and experimented with (eg, a pilot program or test
network), but not that all details have been worked out.

For a Draft to migrate to Active, there is an expectation that the proposal has
been demonstrated, that the change of significant unforseen issues in complete
adoption is low, and that the proposal will be the "new normal" and expected
behavior going forward.


# Decision Making Process
[power]: #power

There exists a [Protocol Working Group (WG)][wg] which makes DEP status
decisions.  Membership is based on unanimous consensus invidation by the
existing WG. WG members can resign at and time, or be ejected by unanimous (by
organization) decision by the other WG members. Members are expected to commit
to active participation for 6 month windows. The WG is expected to respect the
needs and desires of the community as a whole.

For Draft status, at leat one WG member must review the entire proposal in
detail, give feedback, and give informed approval. If no review takes place in
the fixed time window, the default is to close (reject) until a member is
willing to commit to review. Any WG member can request revisions or
clarifications (blocking acceptance until addressed) or veto acceptance if they
agree. A veto can be overridden by unanimous decision of all other WG members
on an organizational affiliation basis (aka: a single organization can not
unanimously veto a proposal).

For Active status, the default is again negative (the proposal remains a
draft). 

Proposals are expected to be open for at least three days (72 hours) for
comment (and longer to accomodate special circumstances, like holidays). Vetos
can happen up to a week after initially being submitted for review, which might
be retroactive if the proposal was accepted early.

For all other status changes, at least one WG member must vouch for or approve
the change. For example, if a Draft was submitted to be Closed, but the WG
decides to switch to Accept instead (!), only one WG member needs to propose
the change. If the WG is deadlocked (eg, conflicting proposals), the default
action is taken (which is no action).


# Rationale
[rationale]: #rationale

This proposal attempts to head off a couple negative patterns.

Proposals could get stuck in an ambiguous indefinite state anywhere along the
procoess, leaving ambiguity about their state. This is mitigated by setting
time limits and default outcomes.

A related possible problem is when something is submitted for formal review,
but changes rapidly based on feedback, distracting reviewers and making it hard
to give clear feedback. Or, an incomplete proposal is submitted, reviewers ask
for more details, then need to re-review when the details arrive. This is
mitigated by setting expectations for the completeness of proposals before
submission, and giving an explicit "withdrawl and resubmit" workflow for larger
changes.

When defining the proposal statuses, there are two manin questions for
technical standards: does draft status mean *could* be implemented, or *has*
been implemented? We chose "could". For active (or "final") status, is the
proposal *expected* to be dominant in the wild, or *is it already* dominant in
the wild? We chose "expected". In both cases we are emphasizing clarification
and stabilization of new ideas, as opposed to enforcing interoperability of
competing formulations of the same idea.

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
  Bittorrent protocol has a lot of similarities to Dat, and as a "protocol" is
  most similar in scope.
* The **[Rust Language RFC Process][rust-rfc]** is relatively new, but has had
  a huge volume of proposals, rivaling even the IETF. The process is relatively
  lightweight and happens entirely on Github; it is the most similar to the DEP
  process proposed here. Rust has strong organizational backing with defined
  leadership roles; proposals are reviewed by specific sub-teams.
* **[IETF RFC Process][ietf]**: perhaps the oldest and best known RFC process,
  under the motto of "rough consensus and working code". The process is very
  bespoke (involving custom file formats and software) and heavy on process
  (with working groups and in-person meetings).
* **[XMPP Standards Process][xmpp]**: has the interesting sub-pattern of
  regularly updated (annual) standards. XMPP is also a protocol, like
  Bittorrent. The protocol was designed for easy extension, and at various
  points has seen adoption, extention, and pressure from powerful entities.
* **Python Enhancement Process** documented in [PEP 1][pep-1]. PEPs are
  relatively broad in scope (they often speak to process and organizational
  dynamics), and are widely cited directly by name. Proposals are usually
  debated in great detail on mailing lists before being proposed. Python has a
  BDFL (benevolent dictator for life) who has final say over proposals, though
  he sometimes delegates to deputies.
* The **W3C** is a paid membership organization which, like the IETF, is made
  up of entities large and small, for-profit and altruistic, with decent
  regional diversity.


[bep-1]: http://bittorrent.org/beps/bep_0001.html
[rust-rfc]: https://github.com/rust-lang/rfcs
[xmpp]: https://xmpp.org/about/standards-process.html
[ietf]: https://www.ietf.org/about/process-docs.html
[pep-1]: https://www.python.org/dev/peps/pep-0001/

# Unresolved questions
[unresolved]: #unresolved-questions

How is Working Group membership defined in the long run? This could be the
topic of a follow-on Process DEP while this DEP is in Draft status..

The intention is to retroactively document the entire Dat protocol in the form
of DEPs, but a specific plan for this hasn't been worked out yet. This is
expected to be tackled by the Working Group while this DEP is in Draft status.

How mutable should Draft Standards DEPs be over time? What about Process DEPs?
Should there be an additional status ("Living"?) for DEPs that are expected to
evolve, or is this against the whole philosophy of having specific stable
documents to reference? This is expected to be decided while this DEP is in
Draft status.

# Changelog
[changelog]: #changelog

- 2018-01-15: TODO: First complete draft submitted for review

