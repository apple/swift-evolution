# Swift Evolution Process

Swift is a powerful and intuitive programming language that is designed to make writing and maintaining correct programs easier. Swift is growing and evolving, guided by a community-driven process referred to as the Swift evolution process, maintained by the [Language Steering Group][language-steering-group]. This document outlines the Swift evolution process and how a feature grows from a rough idea into something that can improve the Swift development experience for millions of programmers.

## Scope

The Swift evolution process covers all design changes, no matter how small, to the Swift language, its standard library, and the core tools necessary to build Swift programs.  This includes additions, removals, and changes to:
- the features of the Swift language,
- the public interface of the Swift standard library,
- the configuration of the Swift compiler, and
- the core tools of the Swift package ecosystem, including the configuration of the [Swift package manager](https://www.swift.org/package-manager/) and the design of its manifest files.

The design of other tools, such as IDEs, debuggers, and documentation generators, is not covered by the evolution process.  The Core Team may create workgroups to guide and make recommendations about the development of these tools, but the output of those workgroups is not reviewed.

The evolution process does not cover experimental features, which can be added, changed, or removed at any time.  Implementors should take steps to prevent the accidental use of experimental features, such as by enabling them only under explicitly experimental options.  Features should not be allowed to remain perpetually experimental; a feature with no clear path for development into an official feature should be removed.

Changes such as bug fixes, optimizations, or diagnostic improvements can be contributed via the normal contribution process; see [Contributing to Swift](https://www.swift.org/contributing/).  Some bug fixes are effectively substantial changes to the design, even if they're just making the implementation match the official documentation; whether such a change requires evolution review is up to the appropriate evolution workgroup.

Which parts of the Swift project are covered by the evolution process is ultimately up to the judgment of the Core Team.

## Goals

The Swift evolution process aims to leverage the collective ideas, insights, and experience of the Swift community to improve the Swift development experience. Its two primary goals are:

* Engage the wider Swift community in the ongoing evolution of Swift, and
* Maintain the vision and conceptual coherence of Swift.

There is a natural tension between these two goals. Open evolution processes are, by nature, chaotic. Yet, maintaining a coherent vision for something as complicated as a programming language requires some level of coordination. The Swift evolution process aims to strike a balance that best serves the Swift community as a whole.

## Community structure

The [Core Team](https://www.swift.org/community/#core-team) is responsible for the strategic direction of Swift.  The Core Team creates workgroups focused on specific parts of the project.  When the Core Team gives a workgroup authority over part of the evolution of the project, that workgroup is called an evolution workgroup.  Evolution workgroups manage the evolution process for proposals under their authority, working together with other workgroups as needed.

Currently, there is only one evolution workgroup:

* The [Language Steering Group][language-steering-group] has authority over the evolution of the Swift language and its standard library.

The Core Team manages (or delegates) the evolution process for proposals outside these areas.  The Core Team also retains the power to override the evolution decisions of workgroups when necessary.

## Proposals, roadmaps, and visions

There are three kinds of documents commonly used in the evolution process.

* An evolution *proposal* describes a specific proposed change in detail.  All evolution changes are advanced as proposals which will be discussed in the community and given a formal open review.

* An evolution *roadmap* describes a concrete plan for how a complex change will be broken into separate proposals that can be individually pitched and reviewed.  Considering large changes in small pieces allows the community to provide more focused feedback about each part of the change.  A roadmap makes this organization easier for community members to understand.

  Roadmaps are planning documents that do not need to be reviewed.

* An evolution *vision* describes a high-level design for a broad topic (for example, string processing or concurrency).  A vision creates a baseline of understanding in the community for future conversations on that topic, setting goals and laying out a possible program of work.

  Visions must be approved by the appropriate evolution workgroup.  This approval is an endorsement of the vision's basic ideas, but not of any of its concrete proposals, which must still be separately developed and reviewed.

## Participation

Everyone is welcome to propose, discuss, and review ideas to improve
the Swift language and standard library in the
[Evolution section of the Swift forums](https://forums.swift.org/c/evolution).
Before posting a review, please see the section "What goes into a review?" below.

## What goes into a review?

The goal of the review process is to improve the proposal under review
through constructive criticism and, eventually, determine the
direction of Swift. When writing your review, here are some questions
you might want to answer in your review:

* What is your evaluation of the proposal?
* Is the problem being addressed significant enough to warrant a change to Swift?
* Does this proposal fit well with the feel and direction of Swift?
* If you have used other languages or libraries with a similar feature, how do you feel that this proposal compares to those?
* How much effort did you put into your review? A glance, a quick reading, or an in-depth study?

Please state explicitly whether you believe that the proposal should be accepted into Swift.

## How to propose a change

1. **Check prior proposals**

   Many ideas come up frequently, and may either be in active discussion on the forums, or may have been discussed already and have joined the [Commonly Rejected Proposals](commonly_proposed.md) list. Please [search the forums](https://forums.swift.org/search) for context before proposing something new.

1. **Consider the goals of the upcoming Swift release**

   Each major Swift release is focused on a [specific set of goals](README.md)
   described early in the release cycle. When proposing a change to
   Swift, please consider how your proposal fits in with the larger goals
   of the upcoming Swift release. Proposals that are clearly out of scope
   for the upcoming Swift release will not be brought up for review. If you can't resist discussing a proposal that you know is out of scope, please include the tag `[Out of scope]` in the subject.

1. **Socialize the idea**

   Propose a rough sketch of the idea in the ["pitches" section of the Swift forums](https://forums.swift.org/c/evolution/pitches), the problems it solves, what the solution looks like, etc., to gauge interest from the community.

1. **Develop the proposal and implementation**

   1. Expand the rough sketch into a formal proposal using the [proposal template](proposal-templates/0000-swift-template.md).
   1. In the [swift-evolution repository][swift-evolution-repo], open a [draft pull request][draft-pr] that adds your proposal to the [proposals directory](/proposals).
   1. Announce the pull request on the forums and edit the root post to link out to the pull request.
   1. Refine the formal proposal in the open as you receive further feedback on the forums or the pull request.
      A ripe proposal is expected to address commentary from present and past
      discussions of the idea.

      Meanwhile, start working on an implementation.
      Prototyping an implementation and its uses *alongside* the formal proposal
      is important because it helps to determine an adequate scope, ensure
      technical feasibility, and validate that the proposal lives up to
      its motivation.

      A pull request with a working implementation is *required* for the
      proposal to be accepted for review.
      Proposals that can ship as part of the [Standard Library Preview package][preview-package]
      should be paired with a pull request against the [swift-evolution-staging repository][swift-evolution-staging].
      All other proposals should be paired with an implementation pull request
      against the [main Swift repository](https://github.com/apple/swift).

      The preview package can accept new types, new protocols, and extensions to
      existing types and protocols that can be implemented without access to
      standard library internals or other non-public features.
      For more information about the kinds of changes that can be implemented in
      the preview package, see [SE-0264](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0264-stdlib-preview-package.md).

1. **Request a review**

   Once you have a working implementation and believe the proposal is sufficiently detailed and clear, mark the draft pull request in the [swift-evolution repository][swift-evolution-repo] as ready for review to indicate to the appropriate evolution workgroup that you would like the proposal to be reviewed.

> [!IMPORTANT]
> In general, and especially [during the review period](#review-process), be responsive to questions and feedback about the proposal.

[draft-pr]: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests#draft-pull-requests

## Review process

The review process for a particular proposal begins when a member of
the appropriate evolution workgroup accepts a pull request of a new or updated proposal into
the [swift-evolution repository][swift-evolution-repo]. That
member becomes the *review manager* for the proposal. The proposal
is assigned a proposal number (if it is a new proposal), and then enters
the review queue. If your proposal's accompanying implementation takes the form of a package, the review manager will merge your pull request into a new branch in the [swift-evolution-staging repository][swift-evolution-staging].

The review manager will work with the proposal authors to schedule the
review. Reviews usually last a single week, but can run longer for
particularly large or complex proposals.

When the scheduled review period arrives, the review manager will post
the proposal to the ["Proposal reviews" section of the Swift forums][proposal-reviews]
with the proposal title and update the list of active
reviews. To avoid delays, it is important that the proposal authors be
available to answer questions, address feedback, and clarify their
intent during the review period.

After the review has completed, the managing evolution workgroup will make a decision on
the proposal. The review manager is responsible for determining
consensus among the workgroup members, then reporting their decision
to the proposal authors and forums. The review manager will
update the proposal's state in the [swift-evolution
repository][swift-evolution-repo] to reflect that decision.

## Proposal states

```mermaid
flowchart LR
  %% <https://mermaid-js.github.io/>

  %% Nodes:
  1{{"Awaiting\nreview"}}
  2{{"Scheduled\nfor review"}}
  3{"Active\nreview"}
  4["Returned\nfor revision"]
  5(["Withdrawn"])
  6(["Rejected"])
  7_8["Accepted\n(with revisions)"]
  9[["Previewing"]]
  10(["Implemented"])

  %% Links:
  1 ==> 3 ==> 7_8 ==> 10
  1 -.-> 2 -.-> 3 -.-> 4 -.-> 5 & 1
  3 -.-> 6
  7_8 -.-> 9 -.-> 10
```

A given proposal can be in one of several states:

* **Awaiting review**: The proposal is awaiting review. Once known, the dates
  for the actual review will be placed in the proposal document. When the review
  period begins, the review manager will update the state to *Active review*.
* **Scheduled for review (...)**: The public review of the proposal
  in the [Swift forums][proposal-reviews]
  has been scheduled for the specified date range.
* **Active review (...)**: The proposal is undergoing public review
  in the [Swift forums][proposal-reviews].
  The review will continue through the specified date range.
* **Returned for revision**: The proposal has been returned from review
  for additional revision to the current draft.
* **Withdrawn**: The proposal has been withdrawn by the original submitter.
* **Rejected**: The proposal has been considered and rejected.
* **Accepted**: The proposal has been accepted and is either awaiting
  implementation or is actively being implemented.
* **Accepted with revisions**: The proposal has been accepted,
  contingent upon the inclusion of one or more revisions.
* **Previewing**: The proposal has been accepted and is available for preview
  in the [Standard Library Preview package][preview-package].
* **Implemented (Swift Next)**:
  The proposal has been implemented (for the specified version of Swift).
  If the proposal's implementation spans multiple version numbers,
  write the version number for which the implementation will be complete.

[swift-evolution-repo]: https://github.com/swiftlang/swift-evolution  "Swift evolution repository"
[swift-evolution-staging]: https://github.com/swiftlang/swift-evolution-staging  "Swift evolution staging repository"
[proposal-reviews]: https://forums.swift.org/c/evolution/proposal-reviews "'Proposal reviews' category of the Swift forums"
[status-page]: https://apple.github.io/swift-evolution/
[preview-package]: https://github.com/apple/swift-standard-library-preview/
[language-steering-group]: https://www.swift.org/language-steering-group

## Review announcement

When a proposal enters review, a new topic will be posted to the ["Proposal Reviews" section of the Swift forums][proposal-reviews]
using the following template:

---

Hello Swift community,

The review of "\<\<PROPOSAL NAME>>" begins now and runs through \<\<REVIEW
END DATE>>. The proposal is available here:

> https://linkToProposal

Reviews are an important part of the Swift evolution process. All review feedback should be either on this forum thread or, if you would like to keep your feedback private, directly to the review manager. When emailing the review manager directly, please keep the proposal link at the top of the message.

##### Trying it out

If you'd like to try this proposal out, you can [download a toolchain supporting it here]().  You will need to add `-enable-experimental-feature FLAGNAME` to your build flags.  \<\<Review managers should revise this section as necessary, or they can delete it if a toolchain is considered unnecessary for this proposal.\>\>

##### What goes into a review?

The goal of the review process is to improve the proposal under review
through constructive criticism and, eventually, determine the direction of
Swift. When writing your review, here are some questions you might want to
answer in your review:

* What is your evaluation of the proposal?
* Is the problem being addressed significant enough to warrant a
  change to Swift?
* Does this proposal fit well with the feel and direction of Swift?
* If you have used other languages or libraries with a similar
  feature, how do you feel that this proposal compares to those?
* How much effort did you put into your review? A glance, a quick
  reading, or an in-depth study?

More information about the Swift evolution process is available at

> <https://github.com/swiftlang/swift-evolution/blob/main/process.md>

Thank you,

-\<\<REVIEW MANAGER NAME>>

Review Manager

---
