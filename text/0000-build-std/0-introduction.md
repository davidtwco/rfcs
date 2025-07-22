- Feature Name: `build-std`
- Start Date: 2025-06-05
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

While Rust's pre-built standard library has proven itself sufficient for the
majority of use cases, there are a handful of use cases that are not well
supported:

1. Rebuilding the standard library to match the user's profile
2. Rebuilding the standard library with ABI-modifying flags
3. Building the standard library for tier three targets

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

### Terminology
[terminology]: #terminology

The following terminology is used throughout the RFC:

- "the standard library" is used to refer to all of the crates that comprise the
  standard library - `core`, `alloc` and `std`
- "std" is used to refer only to the `std` crate, not the entirety of the standard
  library

# Contents
[contents]: #contents

This RFC has the following contents:

1. [Summary][summary] (you are here)

    - Introduction to the proposal, its scope, terminology/conventions used and
      the structure of the RFC
  
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

5. [Appendix II: Exhaustive literature review](./4-appendix-literature-review.md)
    
    - More detailed summaries of the relevant issues, discussions, pull requests
      and proposals that comprise the history of the build-std feature since
      2015
    
    - [*History*](./2-history.md) aims to summarise this content further and
      cover everything that should be necessary to understand the proposal

[davidtwco]: https://github.com/davidtwco
[adamgemmell]: https://github.com/adamgemmell