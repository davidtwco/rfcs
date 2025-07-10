- Feature Name: `build-std`
- Start Date: 2025-06-05
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

<!--

This document has lots of links and anchors and internal references, follow
these rules to keep it consistent:

- Text is wrapped at ~80 characters, except for headings
- Every header has a reference link defined below it with its anchor. Top-level
  sections just match the header text. Other sections have a prefix (e.g.
  "rationale-foo", not "foo").
- Add parentheticals with `([?][anchor])` wherever an explanation is justified,
  linking to the relevant sub-section in the rationale/alterantive section.
- Each justification/alternative in a section must be included in the bullet
  list at the bottom of that sub-section, likewise with unresolved questions and
  future possibilities.
- Each future possibility, unresolved question and rationale/alternative must
  backlink back to the section that links to it.
- Rationale/alternatives must be in the order that they are referenced in the
  text, including in end-of-sub-section lists.
- Use the passive voice.
- Ensure that Appendix I is up-to-date after any changes
- Appendix II should only reflect the discussion on the cited sources, rather
  than the current status quo if it has changed

-->

# Summary
[summary]: #summary

While Rust's pre-built standard library has proven itself sufficient for the
majority of use cases, there are a handful of use cases that are not well
supported:

1. Rebuilding the standard library to match the user's profile
2. Rebuilding the standard library with ABI-modifying flags
3. Building the standard library for tier three targets

This RFC proposes a handful of changes to Cargo, the compiler and standard
library with the goal of defining a minimal build-std that has the potential of
being stabilised:

- Explicitly declaring support for the standard library in target specs
- Explicit and implicit dependencies on the standard library in `Cargo.toml`
- Re-building the standard library when the profile or target modifiers change

This RFC is co-authored by [David Wood][davidtwco] and
[Adam Gemmell][adamgemmell]. To improve the readability of this RFC, it does not
follow the standard RFC template, while still aiming to capture all of the
salient details that the template encourages.

### Scope
[scope]: #scope

build-std, as proposed by this RFC, has many restrictions and limitations that
mean it will not support most use cases that those waiting for build-std hope
that it will. This is an explicit and deliberate choice. This RFC will focus on
resolving the key questions that will enable a MVP of build-std to be accepted
and stabilised. This will lay the foundation for future proposals to lift
restrictions and enable build-std to support more use cases, without those
proposals having to survey the ten+ years of issues, pull requests and
discussion that this RFC has.

### Acknowledgements
[acknowledgements]: #acknowledgements

This RFC would not have been possible without the advice, feedback and support
of [Josh Triplett][joshtriplett], [Eric Huss][ehuss],
[Wesley Wiser][wesleywiser] and [Tomas Sedovic][tomassedovic]. Thanks to
[mati865][mati865] for advising on some of the specifics related to special
object files, [petrochenkov][petrochenkov] for his expertise on rustc's
dependency loading and name resolution and to [Ed Page][epage] for writing about
opaque dependencies.

### Terminology
[terminology]: #terminology

The following terminology is used throughout the RFC:

- "the standard library" is used to refer to all of the crates that comprise the
  standard library - `core`, `alloc` and `std`
- "std" is used to refer only to the `std` crate, not the entirety of the standard
  library

Throughout the RFC's [*Detailed explanation*][detailed-explanation],
parentheticals with "?" links will be present that which connect the relevant
section in the [*Rationale and alternatives*][rationale-and-alternatives] to
justify a decision or provide alternatives to it.

Additionally, "note alerts" will be used in the
[*Detailed explanation*][detailed-explanation] section to separate
implementation considerations from the core proposal. Implementation detail
should be considered non-normative. These details could change during
implementation and are present solely to demonstrate that the implementation
feasibility has been considered and to provide an example of how implementation
could proceed.

> [!NOTE]
>
> This is an example of a "note alert" that will be used to separate
> implementation detail from the proposal proper.

# Contents
[contents]: #contents

Due to the length of this RFC, to ease reviewing on GitHub, it is split over
multiple files:

1. [Summary][summary] (you are here)

    - Introduction to the proposal, its scope, terminology/conventions used and
      the structure of the RFC
  
2. [Background](./1-background.md)

    - Detailed explanations relevant and impacted parts of the Rust toolchain
      currently work

3. [History](./2-history.md)

    - Chronological summary of the various proposals and discussions around the
      ability to rebuild the standard library, and of the current experimental
      implementation in Cargo

4. [Motivation](./3-motivation.md)

    - Chronological summary of the various proposals and discussions around the
      ability to rebuild the standard library, and of the current experimental
      implementation in Cargo

5. [Proposal](./4-proposal.md)

    - [Detailed explanation][detailed-explanation]

        - Proposal for an implementation of build-std in the Rust toolchain

    - [Rationale and alternatives][rationale-and-alternatives]
      
      - Justifications for each of the decisions made in the detailed
        explanation and exploration of the alternatives to those decisions

    - [Unresolved questions](./4-proposal.md#unresolved-questions)
    
      - Questions left unanswered by this RFC

    - [Future possibilities](./4-proposal.md#future-possibilities)
  
        - Future proposals made possible by this RFC and anticipated follow-up
          work

6. [Appendix I: Summary of changes](./5-appendix-summary-of-changes.md)
    
    - Summary of each of the changes from [*Proposal*](./4-proposal.md) which
      would need implemented in the Rust toolchain, organised by responsible
      project team

7. [Appendix II: Exhaustive literature review](./6-appendix-literature-review.md)
    
    - More detailed summaries of the relevant issues, discussions, pull requests
      and proposals that comprise the history of the build-std feature since
      2015
    
    - [*History*](./2-history.md) aims to summarise this content further and
      cover everything that should be necessary to understand the proposal

[detailed-explanation]: ./4-proposal.md
[rationale-and-alternatives]: ./4-proposal.md#rationale-and-alternatives

[davidtwco]: https://github.com/davidtwco
[adamgemmell]: https://github.com/adamgemmell
[ehuss]: https://github.com/ehuss
[epage]: https://github.com/epage
[joshtriplett]: https://github.com/joshtriplett
[mati865]: https://github.com/mati865
[petrochenkov]: https://github.com/petrochenkov
[tomassedovic]: https://github.com/tomassedovic
[wesleywiser]: https://github.com/wesleywiser
