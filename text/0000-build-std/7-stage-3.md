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
  > rustc's compatibility checking from [Stage 2][stage2] will be extended to
  > allow checking for any mismatch in relevant compilation flags (e.g.
  > excluding things like dependency rlib search paths which will necessarily
  > differ). rustc will not be able to serialise the value of each flag into
  > rlib metadata due to performance overhead but will be able to serialise and
  > compare a hash of these values.

The above examples apply to any other profile too, such as `bench` or `test`.
These options are primarily useful for users wanting to use the same codegen
flags with the standard library or have a more debuggable standard library.

As with the "always" option, the exact crates from the standard library to be
built are determined by the `build-std-crate` option or explicit dependencies on
the standard library if [*Stage 1b*][stage1b] was implemented.

*See the following sections for rationale/alternatives:*

- [*Why permit `build-std` in `[profile]`?*][rationale-profile]
- [*Why does `[profile]` have higher precedence than `[build]` and lower than `[target]`?*][rationale-profile-precedence]
- [*Why have "match-profile" as the default for the release profile?*][rationale-default]
- [*Why not always default to "match-profile"?*][rationale-why-not-always-rebuild]
- [*Why add "compatible-profile"?*][rationale-compatible-profile]
- [*Why does "compatible-profile" use the standard library's profiles?*][rationale-compatible-profile-std]
- [*Why add "match-profile"?*][rationale-match-profile]

*See the following sections for relevant unresolved questions:*

- [*What should the "match-profile" and "compatible-profile" values of `build-std` be named?*][unresolved-naming]
- [*Should `build-std` be in `[profile]` if it only makes in the Cargo configuration `[profile]`?*][unresolved-profile]

## Stability guarantees
[stability-guarantees]: #stability-guarantees

build-std enables a much greater array of configurations of the standard library
to exist and be produced by stable toolchains than the single configuration that
is distributed today.

It is not feasible for the Rust project to test every combination of profile
configuration, Cargo feature, target and standard library crate. As such, the
stability of build-std as a mechanism must be separated from the stability
guarantees which apply to configurations of the standard library it enables.

For example, while a stable build-std mechanism may permit the standard library
to be built for a tier three target, the Rust project continues to make no
commitments or guarantees that the standard library for that target will
function correctly or build at all. Even on a tier one target, the Rust project
cannot test every possible variation of the standard library that build-std
enables.

The tier of a target no longer determines whether the availability of the
standard library, but rather the level of support provided for the standard
library on the target.

Cargo and Rust project documentation will clearly document the configurations
which are tested upstream and are guaranteed to work. Any other configurations
are supported on a strictly best-effort basis. The Rust project may later choose
to provide more guarantees for some well-tested configurations (e.g. enabling
sanitisers).

There are also no guarantees about the exact configuration of the standard
library. Over time, the standard library built by build-std could be changed to
be closer to that of the pre-built standard library.

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
turn more narrowly scoped than the global default in `[build]`.

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

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The following small details are likely to be bikeshed prior to Stage 3 acceptance or
stabilisation and aren't pertinent to the overall design:

## What should the "match-profile" and "compatible-profile" values of `build-std` be named?
[unresolved-naming]: #what-should-the-match-profile-and-compatible-profile-values-of-build-std-be-named

It could be named something else.

## Should `build-std` be in `[profile]` if it only makes in the Cargo configuration `[profile]`?
[unresolved-profile]: #should-build-std-be-in-profile-if-it-only-makes-in-the-cargo-configuration-profile

This could be unintuitive for users.

# Future possibilities
[future-possibilities]: #future-possibilities

There are not currently any documented follow-ups to Stage 3.

[stage1b]: ./5-stage-1b.md#proposal
[stage2]: ./6-stage-2.md