- Feature Name: `build-std-match-profile`
- Start Date: 2025-06-05
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

<!-- This RFC will be submitted separately from build-std RFC after the it is accepted -->

# Summary
[summary]: #summary

This RFC proposes extending the `build-std` option with new values which
automatically rebuild the standard library to match the user's current profile.

**This RFC is is part of the [build-std project goal] and a series of build-std
RFCs:**

1. build-std context ([rfcs#3873])
    - [Background][rfcs#3873-background]
    - [History][rfcs#3873-history]
    - [Motivation][rfcs#3873-motivation]
2. `build-std="always"` ([rfcs#3874])
    - [Proposal][rfcs#3874-proposal]
    - [Rationale and alternatives][rfcs#3874-rationale-and-alternatives]
    - [Unresolved questions][rfcs#3874-unresolved-questions]
    - [Future possibilities][rfcs#3874-future-possibilities]
3. Explicit standard library dependencies ([rfcs#3875])
    - [Proposal][rfcs#3875-proposal]
    - [Rationale and alternatives][rfcs#3875-rationale-and-alternatives]
    - [Unresolved questions][rfcs#3875-unresolved-questions]
    - [Future possibilities][rfcs#3875-future-possibilities]
4. `build-std="compatible"` ([rfcs#XXXX])
    - [Proposal][rfcs#XXXX-proposal]
    - [Rationale and alternatives][rfcs#XXXX-rationale-and-alternatives]
    - [Unresolved questions][rfcs#XXXX-unresolved-questions]
    - [Future possibilities][rfcs#XXXX-future-possibilities]
6. `build-std="match-profile"` (this RFC)
    - [Proposal][proposal]
    - [Rationale and alternatives][rationale-and-alternatives]
    - [Unresolved questions][unresolved-questions]
    - [Future possibilities][future-possibilities]

# Motivation
[motivation]: #motivation

This RFC builds on a large collection of prior art collated in the
[`build-std-context`][rfcs#3873] RFC, and is aimed at at allowing users to
rebuild the standard library with different codegen flags or profile as
identified in its [*Motivation*][rfcs#3873-motivation] section.

# Proposal
[proposal]: #proposal

Cargo will also permit `build-std` to be set as part of the `[profile]` section
([?][rationale-profile]). `build-std` configuration locations precedence is
updated as follows ([?][rationale-profile-precedence]):

1. `[target.<triple>]`
2. `[target.<cfg>]`
3. `[profile]`
4. `[build]`

The `build-std` option in the Cargo configuration will be extended with two new
values - "compatible-profile" or "match-profile":

```toml
[build]
build-std = "match-profile" # or `compatible-profile`/`compatible`/`always`/`never`
```

"match-profile" will become the default value for the release profile
([?][rationale-default], [?][rationale-why-not-always-rebuild]). The `bench`
profile inherits this default from `release`.

Like `build-std = "compatible"`, when set to either "compatible-profile" or
"match-profile", then the standard library crates will be rebuilt
automatically when a pre-built standard library is not present.

- "compatible-profile" rebuilds the standard library if the user is using a
  different profile than the default "release" profile of the pre-built standard
  library or a rebuild is necessary to maintain compatibility with the user's
  crate ([?][rationale-compatible-profile]).

  Cargo will build the standard library using the same profile as the user, as
  defined in the standard library workspace
  ([?][rationale-compatible-profile-std]). It will vary only in the target modifiers
  necessary to maintain compatibility with the user's crates.

  Pre-built available? | User's profile | Target modifiers changed? | Standard library re-built?
  -------------------- | -------------- | ------------------------- | --------------------------
  No                   | `dev`          | N/A                       | Yes, std's `dev`
  No                   | `release`      | N/A                       | Yes, std's `release`
  Yes                  | `dev`          | N/A                       | Yes, std's `dev`
  Yes                  | `release`      | Unchanged                 | No
  Yes                  | `release`      | Changed                   | Yes, std's `release`

- "match-profile" rebuilds the standard library using the configuration of the
  user's current profile ([?][rationale-match-profile]). Cargo will check
  whether the compilation flags it would intend to use for the standard library
  match those used with the pre-built standard library by asking rustc.

  Pre-built available? | User's profile | Target modifiers changed? | Standard library re-built?
  -------------------- | -------------- | ------------------------- | --------------------------
  No                   | `dev`          | N/A                       | Yes, user's `dev`
  No                   | `release`      | N/A                       | Yes, user's `release`
  Yes                  | `dev`          | N/A                       | Yes, user's `dev`
  Yes                  | `release`      | Unchanged                 | Yes, user's `release`
  Yes                  | `release`      | Changed                   | Yes, user's `release`

  > [!NOTE]
  >
  > rustc's compatibility checking from [`build-std = "compatible"`][rfcs#XXXX]
  > will be extended to allow checking for any mismatch in relevant compilation
  > flags (e.g. excluding things like dependency rlib search paths which will
  > necessarily differ). rustc will not be able to serialise the value of each
  > flag into rlib metadata due to performance overhead but will be able to
  > serialise and compare a hash of these values.

The above examples apply to any other profile too, such as `bench` or `test`.
When custom profiles are used, the standard library will be built in the profile
that the custom profile ultimately inherited from (via `inherited-from`). These
options are primarily useful for users wanting to use the same codegen flags
with the standard library or have a more debuggable standard library.

As with the "always" option, the exact crates from the standard library to be
built are determined by the `build-std-crates` option or explicit dependencies
on the standard library if [*Standard library dependencies*][rfcs#3875] were
implemented.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This section aims to justify all of the decisions made in the proposed design
from [*Proposal*][proposal] and discuss why alternatives were not chosen.

## Why permit `build-std` in `[profile]`?
[rationale-profile]: #why-permit-build-std-in-profile

Configurations like "match-profile" for `build-std` make most sense when
combined with Cargo profiles that aim to maximise the optimisation of the final
binary. It is more likely that users would want to use "match-profile" with the
release profile than by default (as in `[build]`) or for a specific target (as
in `[target]`).

However, permitting `build-std` in `[profile]` when in Cargo configurations, but
not in Cargo manifests, is inconsistent with other options that exist in
profiles.

↩ [*Proposal*][proposal]

## Why does `[profile]` have higher precedence than `[build]` and lower than `[target]`?
[rationale-profile-precedence]: #why-does-profile-have-higher-precedence-than-build-and-lower-than-target

`[target]` configuration is more narrowly scoped than `[profile]` which is in
turn more narrowly scoped than the global default in `[build]`. There is no
existing precedent in Cargo for these sections having the precedence currently
proposed.

↩ [*Proposal*][proposal]

## Why have "match-profile" as the default for the release profile?
[rationale-default]: #why-have-match-profile-as-the-default-for-the-release-profile

`build-std = "match-profile"` is intended to be used when additional time
spent building the standard library is not a problem and the quality of the
final artifact is paramount. This corresponds closely with the release profile,
where additional time spent on optimisations (e.g. with `-Ctarget-cpu`) is
acceptable. Always re-building the standard library with the user's profile
configuration in release mode is likely to result in a more optimised build than
with the pre-built standard library and is thus a reasonable default for the
`release` and `bench` profiles.

↩ [*Proposal*][proposal]

### Why not always default to "match-profile"?
[rationale-why-not-always-rebuild]: #why-not-always-default-to-match-profile

Cargo's users don't currently expect that changing any part of their profile
configuration, such as trying a different optimisation level, would trigger a
rebuild of the standard library. For small projects, rebuilding the standard
library could be a significant increase in the overall build time for a project.

If `build-std = "match-profile"` were the default, the standard library
could be rebuilt quite frequently without much benefit. Especially as the
pre-built standard library is built using the release profile, all debug profile
builds would immediately trigger a rebuild of the standard library.

↩ [*Proposal*][proposal]

### Why add "compatible-profile"?
[rationale-compatible-profile]: #why-add-compatible-profile

"compatible-profile" is useful for when users want a more debuggable standard
library while keeping rebuilds of the standard library to a minimum.

↩ [*Proposal*][proposal]

### Why does "compatible-profile" use the standard library's profiles?
[rationale-compatible-profile-std]: #why-does-compatible-profile-use-the-standard-librarys-profiles

By using the standard library's profile definitions, the library team will be
able to define a "dev" profile that is most useful for the standard library.

↩ [*Proposal*][proposal]

### Why add "match-profile"?
[rationale-match-profile]: #why-add-match-profile

"match-profile" is useful for rebuilding the standard library with the same
codegen flags as the rest of the user's project, such as using `-Ctarget-cpu` to
gain additional optimisations.

↩ [*Proposal*][proposal]

# Prior art
[prior-art]: #prior-art

See the [*Background*][rfcs#3873-background] and [*History*][rfcs#3873-history]
of the build-std context RFC.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The following small details are likely to be bikeshed prior to RFC acceptance or
stabilisation and aren't pertinent to the overall design:

## What should the "match-profile" and "compatible-profile" values of `build-std` be named?
[unresolved-naming]: #what-should-the-match-profile-and-compatible-profile-values-of-build-std-be-named

It could be named something else.

## Should `build-std` be in `[profile]` if it only makes in the Cargo configuration `[profile]`?
[unresolved-profile]: #should-build-std-be-in-profile-if-it-only-makes-in-the-cargo-configuration-profile

This could be unintuitive for users.

# Future possibilities
[future-possibilities]: #future-possibilities

There are not currently any documented follow-ups to this RFC.

[build-std project goal]: https://rust-lang.github.io/rust-project-goals/2025h2/build-std.html
[rfcs#3873]: https://rust-lang.github.io/rfcs/3873-build-std-context.html
[rfcs#3873-background]: https://rust-lang.github.io/rfcs/3873-build-std-context.html#background
[rfcs#3873-history]: https://rust-lang.github.io/rfcs/3873-build-std-context.html#history
[rfcs#3873-motivation]: https://rust-lang.github.io/rfcs/3873-build-std-context.html#motivation
[rfcs#3874]: https://rust-lang.github.io/rfcs/3874-build-std-always.html
[rfcs#3874-proposal]: https://rust-lang.github.io/rfcs/3874-build-std-always.html#proposal
[rfcs#3874-rationale-and-alternatives]: https://rust-lang.github.io/rfcs/3874-build-std-always.html#rationale-and-alternatives
[rfcs#3874-unresolved-questions]: https://rust-lang.github.io/rfcs/3874-build-std-always.html#unresolved-questions
[rfcs#3874-future-possibilities]: https://rust-lang.github.io/rfcs/3874-build-std-always.html#future-possibilities
[rfcs#3875]: https://rust-lang.github.io/rfcs/3875-build-std-explicit-dependencies.html
[rfcs#3875-proposal]: https://rust-lang.github.io/rfcs/3875-build-std-explicit-dependencies.html#proposal
[rfcs#3875-rationale-and-alternatives]: https://rust-lang.github.io/rfcs/3875-build-std-explicit-dependencies.html#rationale-and-alternatives
[rfcs#3875-unresolved-questions]: https://rust-lang.github.io/rfcs/3875-build-std-explicit-dependencies.html#unresolved-questions
[rfcs#3875-future-possibilities]: https://rust-lang.github.io/rfcs/3875-build-std-explicit-dependencies.html#future-possibilities
[rfcs#XXXX]: https://github.com/davidtwco/rfcs/blob/build-std-part-four-compatible/text/0000-build-std-compatible.md
[rfcs#XXXX-proposal]: https://github.com/davidtwco/rfcs/blob/build-std-part-four-compatible/text/0000-build-std-compatible.md#proposal
[rfcs#XXXX-rationale-and-alternatives]: https://github.com/davidtwco/rfcs/blob/build-std-part-four-compatible/text/0000-build-std-compatible.md#rationale-and-alternatives
[rfcs#XXXX-unresolved-questions]: https://github.com/davidtwco/rfcs/blob/build-std-part-four-compatible/text/0000-build-std-compatible.md#unresolved-questions
[rfcs#XXXX-future-possibilities]: https://github.com/davidtwco/rfcs/blob/build-std-part-four-compatible/text/0000-build-std-compatible.md#future-possibilities