- Feature Name: `build-std`
- Start Date: 2025-06-05
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

<!--

This document is long and has lots of authors, follow these rules to maintain a
consistent voice and structure:

Writing style:

- Text is wrapped at ~80 characters, except for headings

- Use the passive voice

- Items in bullet point lists shouldn't end with a period

- Avoid introducing sections that only include other sections and no written
  content within them

- In the proposals, write as if the feature has already been accepted and
  implemented

- Leave a line between each bullet point

- With the exception of table of contents-style sections (in this document and
  in Appendix I), always use reference-style links

- Use a spell checker and broken link checker

- Use 4 spaces for nested bullets, etc.

Structure:

- Every header has a reference link defined below it with its anchor. Top-level
  sections just match the header text. Other sections have a prefix (e.g.
  "rationale-foo", not "foo")

- Add parentheses with `([?][anchor])` wherever an explanation is justified,
  linking to the relevant sub-section in the rationale/alternative section. This
  is not required for unresolved questions and future possibilities

- Each justification/alternative in a section must be included in the bullet
  list at the bottom of that sub-section, likewise with unresolved questions and
  future possibilities

- Each future possibility, unresolved question and rationale/alternative must
  backlink back to the section that links to it

- Rationale/alternatives must be in the order that they are referenced in the
  text, including in end-of-sub-section lists

- Add anyone who has provided feedback prior to the publication of the RFC to
  the acknowledgements section

Consistency:

- Ensure that Appendix I is up-to-date after any changes

- Appendix II should only reflect the discussion on the cited sources, rather
  than the current status quo if it has changed

- build-std should not be in backticks (i.e. not `build-std`)

- "pre-built" should always have a hyphen

- Crate names, Cargo configuration options, compiler flags, file names and
  environment variables should always be in a backticks (e.g. `build-std`)

- Values passed to compiler flags should be in double quotes (e.g. "compatible")

Git:

- Try to keep each individual change to a single commit and describe that change
  in the commit message

  - This makes it easier to review and catch-up

- Keep the first line of commit messages limited to 50 characters and the
  remaining lines to 74 characters

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
salient details that the template encourages. Due to the length of this RFC, it
is split over multiple files to avoid rendering issues and slow loading on some
platforms.

### Scope
[scope]: #scope

build-std, as proposed by this RFC, has many restrictions and limitations that
mean it will not support most use cases that those waiting for build-std hope
that it will. This is an explicit and deliberate choice.

This RFC will focus on resolving the key questions that will enable a MVP of
build-std to be accepted and stabilised. This will lay the foundation for future
proposals to lift restrictions and enable build-std to support more use cases,
without those proposals having to survey the ten+ years of issues, pull requests
and discussion that this RFC has.

As a general rule, this RFC tries to answer the question "what crates of the
standard library get built and when do they get built" and considers anything
else as likely out-of-scope.

### Acknowledgements
[acknowledgements]: #acknowledgements

This RFC would not have been possible without the advice, feedback and support
of [Josh Triplett][joshtriplett], [Eric Huss][ehuss],
[Wesley Wiser][wesleywiser] and [Tomas Sedovic][tomassedovic].

Thanks to [mati865] for advising on some of the specifics related to special
object files, [petrochenkov] for his expertise on rustc's dependency loading and
name resolution; [fee1-dead] for their early and thorough reviews and to
[Ed Page][epage] for writing about opaque dependencies.

Thanks to [Jacob Bramley][jacobbramley] for their feedback on early drafts.

### Terminology
[terminology]: #terminology

The following terminology is used throughout the RFC:

- "the standard library" is used to refer to multiple of the crates that
  constitute the standard library such as `core`, `alloc`, `std`, `test`,
  `proc_macro` or their dependencies.
- "std" is used to refer only to the `std` crate, not the entirety of the
  standard library

Throughout the RFC's "Proposal" sections, parentheses with "?" links will be
present that which link the relevant section in the appropriate "Rationale and
alternatives" section to justify a decision or provide alternatives to it.

Additionally, "note alerts" will be used in the *Proposal* sections to separate
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

This RFC has been split into multiple stages. Each stage is a self-contained
proposal building on the previous and aim to have value independent of later
stages. As such, stages should be able to be accepted, implemented and
stabilised sequentially. of other stages.

As build-std is a complex feature with many interdependent design decisions, it
is challenging to draft a proposal that is small enough to have an achievable
scope in the short-to-medium term while making a convincing argument that it is
forward-compatible with any desired future extensions. A staged proposal enables
this - each stage can have a small and achievable scope, while still allowing a
reviewer to skip ahead and get a sense of what is planned and how that builds on
what came before.

Later stages may be less detailed and complete than the previous stages and
serve to to indicate the direction that build-std will take and help provide
context for the proposals of earlier stages.

1. [Summary][summary] (you are here)

    - Introduction to the proposal, its scope, terminology/conventions used and
      the structure of the RFC

    - [Proposal-wide rationale and alternatives][rationale-and-alternatives]

2. [Background](./1-background.md)

    - Detailed explanations of how relevant and impacted parts of the Rust
      toolchain currently work

3. [History](./2-history.md)

    - Chronological summary of the various proposals and discussions that have
      taken place relating to the ability to rebuild the standard library, and
      of the current experimental implementation in Cargo

4. [Motivation](./3-motivation.md)

    - Descriptions of the varied problems that build-std has been proposed as a
      solution to

5. [Stage 1a: build-std=always](./4-stage-1a.md)

    - Proposes adding a `build-std = "always|never"` option to the Cargo
      configuration which will unconditionally re-build the standard library
      crates listed in a new `build-std-crates` option

    - [Proposal](./4-stage-1a.md#proposal)

    - [Rationale and alternatives](./4-stage-1a.md#rationale-and-alternatives)

    - [Unresolved questions](./4-stage-1a.md#unresolved-questions)

    - [Future possibilities](./4-stage-1a.md#future-possibilities)

6. [Stage 1b: Explicit dependencies](./5-stage-1b.md)

    - Proposes supporting explicit dependencies on the standard library crates in
      `Cargo.toml`

      - Enables Cargo to determine which standard library crates are required by
        the crate graph without `build-std-crates` being set

      - Necessary for future extensions which support public/private standard
        library dependencies or enabling features of the standard library

    - [Proposal](./5-stage-1b.md#proposal)

    - [Rationale and alternatives](./5-stage-1b.md#rationale-and-alternatives)

    - [Unresolved questions](./5-stage-1b.md#unresolved-questions)

    - [Future possibilities](./5-stage-1b.md#future-possibilities)

7. [Stage 2: build-std=compatible](./6-stage-2.md)

    - Proposes extending the `build-std` option with a new `compatible` value
      which will become the default and automatically rebuilds the standard
      library when it is necessary to maintain compatibility with the compiler
      flags used by the rest of the crate graph.

    - [Proposal](./6-stage-2.md#proposal)

    - [Rationale and alternatives](./6-stage-2.md#rationale-and-alternatives)

    - [Unresolved questions](./6-stage-2.md#unresolved-questions)

    - [Future possibilities](./6-stage-2.md#future-possibilities)

8. [Stage 3: build-std=match-profile](./7-stage-3.md)

    - Proposes extending the `build-std` option with new values which
      automatically rebuild the standard library to match the user's current
      profile.

    - [Proposal](./7-stage-3.md#proposal)

    - [Rationale and alternatives](./7-stage-3.md#rationale-and-alternatives)

    - [Unresolved questions](./7-stage-3.md#unresolved-questions)

    - [Future possibilities](./7-stage-3.md#future-possibilities)

9.  [Appendix I: Summary of changes](./8-appendix-summary-of-changes.md)

    - Summary of each of the changes from each stage which would need implemented
      in the Rust toolchain, grouped by the project team whose purview the change
      would fall under

10. [Appendix II: Exhaustive literature review](./9-appendix-literature-review.md)

    - More detailed summaries of the relevant issues, discussions, pull requests
      and proposals that comprise the history of the build-std feature since
      2015

    - [*History*](./2-history.md) aims to summarise this content further and
      cover everything that should be necessary to understand the proposal

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

These rationales and alternatives apply to the proposal as-a-whole, rather than
any specific stage:

## Why not do nothing?
[rationale-why-not-do-nothing]: #why-not-do-nothing

Support for rebuilding the standard library is a long-standing feature request
from subsets of the Rust community and blocks the work of some project teams
(e.g. sanitisers and branch protection in the compiler team, amongst others).
Inaction forces these users to remain on nightly and depend on the unstable
`-Zbuild-std` flag indefinitely. RFCs and discussion dating back to the first
stable release of the language demonstrate the longevity of build-std as a
need.

## Shouldn't build-std be part of rustup?
[rationale-in-rustup]: #shouldnt-build-std-be-part-of-rustup

build-std is effectively creating a new sysroot with a customised standard
library. rustup as Rust's toolchain manager has existing machinery to create and
maintain sysroots, and if it could invoke Cargo to build the standard library
then it could create a new toolchain from a build from a `rust-src` component.
rustup would be invoking tools from the next layer of abstraction (Cargo) in the
same way that Cargo invokes tools from the layer of abstraction after it
(rustc).

A brief prototype of this idea was created and a
[short design document was drafted][why-not-rustup] before concluding that it
would not be possible. With Cargo's artifact dependencies it may be desirable
to build with a different standard library and if rustup was creating different
toolchains per-customised standard library then Cargo would need to have
knowledge of these to switch between them, which isn't possible (and something
of a layering violation). It is also unclear how Cargo would find and use the
uncustomized host sysroot for build scripts and procedural macros. In addition
rustup's knowledge of sysroots and toolchains is limited to the archives it
unpacks - it becoming a part of the build system is not trivial, especially
considering it uses a different versioning system to Cargo, Rust and the
standard library.

[davidtwco]: https://github.com/davidtwco
[adamgemmell]: https://github.com/adamgemmell
[ehuss]: https://github.com/ehuss
[epage]: https://github.com/epage
[fee1-dead]: https://github.com/fee1-dead
[jacobbramley]: https://github.com/jacobbramley
[joshtriplett]: https://github.com/joshtriplett
[mati865]: https://github.com/mati865
[petrochenkov]: https://github.com/petrochenkov
[tomassedovic]: https://github.com/tomassedovic
[wesleywiser]: https://github.com/wesleywiser

[why-not-rustup]: https://hackmd.io/@davidtwco/rkYRlKv_1x
