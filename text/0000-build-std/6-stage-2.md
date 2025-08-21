# Stage 2: `build-std=compatible`

This stage proposes extending the `build-std` option with a new `compatible`
value, which will become the default and automatically rebuild the standard
library when it is necessary to maintain compatibility with the compiler flags
used by the rest of the crate graph.

This is aimed at unblocking the stabilisation of ABI-modifying compiler flags
(as per [motivations](./3-motivation.md)).

# Proposal
[proposal]: #proposal

The `build-std` option in the Cargo configuration will be extended with a new
value:

```toml
[build]
build-std = "compatible" # or `always`/`never`
```

"compatible" will become the default value for `build-std` ([?][rationale-default]).

When `build-std` is set to "compatible", then the standard library crates will
be rebuilt automatically ([?][rationale-why-automatic]) when a pre-built
standard library is not present or the pre-built standard library is
incompatible with the rest of the crate (due to use of target modifiers).

> [!NOTE]
>
> rustc will be extended with the ability to determine whether an existing
> `rlib` artefact is compatible with the flags passed to rustc
> ([?][rationale-rustc-support]).
>
> ```shell-session
> $ rustc -Zreg-struct-return core/lib.rs -o libcore.rlib
> $ rustc --emit compatibility $cargo_flags libcore.rlib
> error: mixing `-Zreg-struct-return` will cause an ABI mismatch in crate `core`
> ```
>
> In the above example, rustc compiles libcore with the `-Zreg-struct-return`
> target modifier to create an `rlib`, and then is invoked to check the
> compatibility of the flags Cargo would have passed to the crate against those
> used with `libcore.rlib`. In this instance, the `rlib` was compiled with
> `-Zreg-struct-return` and this example assumes that `$cargo_flags` does not
> pass this flag, so rustc reports a mismatch.
>
> Cargo can use this mechanism to determine whether it needs to rebuild the
> standard library to ensure compatibility.

The standard library will be rebuilt in its release profile and will only vary
in the target modifier flags necessarily for it to be compatible
([?][rationale-release-profile]).

Pre-built available? | User's profile | Target modifiers changed? | Standard library re-built?
-------------------- | -------------- | ------------------------- | --------------------------
No                   | N/A            | N/A                       | Yes, std's `release`
Yes                  | `dev`          | Unchanged                 | Yes, std's `release`
Yes                  | `release`      | Unchanged                 | No
Yes                  | `release`      | Changed                   | Yes, std's `release`

*It is assumed that changing a target modifier would be part of Cargo profiles,
hence why there is no row for "target modifiers changed, profile unchanged".*

All of rustc's target modifiers are unstable as they cannot be used without
build-std, so Cargo does not expose any configuration for target modifiers
currently.

As with the "always" option, the exact crates from the standard library to be
built are determined by the `build-std-crate` option or explicit dependencies on
the standard library if [*Stage 1b*][stage1b] was implemented.

Multi-target projects (resulting from the "target" field in artifact
dependencies or the use of `per-pkg-target` fields) results in the decision to
rebuild the standard library being made multiple times - once for each target in
the project.

As Cargo does not have per-target profiles nor a way to change the standard
library's profile on a per-target basis, the only way to configure the standard
library differently for different targets is with the use of the `[target]`
sections in the Cargo config.

*See the following sections for rationale/alternatives:*

- [*Why default to "compatible"?*][rationale-default]
- [*Why rebuild the standard library automatically?*][rationale-why-automatic]
- [*Why add compatibility checks in rustc to support build-std?*][rationale-rustc-support]
- [*Why build in release profile?*][rationale-release-profile]

*See the following sections for relevant unresolved questions:*

- [*What should the "compatible" value of `build-std` be named?*][unresolved-compatible]
- [*What should the interface to rustc compatibility checking be?*][unresolved-rustc-compat-interface]

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This section aims to justify all of the decisions made in the proposed design
from [*Proposal*][proposal] and discuss why alternatives were not chosen.

## Why default to "compatible"?
[rationale-default]: #why-default-to-compatible

"compatible" will only trigger when it is necessary to ensure that a user's
current configuration will build successfully. It shouldn't trigger rebuilds for
the user or change the standard library they are using if it isn't strictly
required.

If the default where not "compatible" then when the user enabled a target
modifier through Cargo (when such options are exposed), the user would
immediately face a compilation error and need to go learn about
`build-std = "compatible"` anyway.

↩ [*Proposal*][proposal]

## Why rebuild the standard library automatically?
[rationale-why-automatic]: #why-rebuild-the-standard-library-automatically

Rebuilds of the standard library happening transparently reduce the requirement
that users learn about build-std as something to enable and configure. Combined
with explicit dependencies on the standard library crates from
[Stage 1b][stage1b] or `build-std-crate` from [Stage 1a][stage1a], build-std can
avoid any cost on users that do not require it (by triggering automatically when
a target modifier is changed, and having no unnecessary rebuilds otherwise).

↩ [*Proposal*][proposal]

## Why add compatibility checks in rustc to support build-std?
[rationale-rustc-support]: #why-add-compatibility-checks-in-rustc-to-support-build-std

Without support from rustc, Cargo would need to assume that the pre-built
standard library's configuration is entirely configured in its Cargo profile and
would need to compare the profile of the standard library with the profile of
the user's crate and know which rustc flags are target modifiers.

↩ [*Proposal*][proposal]

## Why build in release profile?
[rationale-release-profile]: #why-build-in-release-profile

As in
[Stage 1a's *"Why does "always" rebuild in release profile?"*][stage1a-why-release],
building in the release profile minimises the differences between a newly-built
std and pre-built std, keeping the implications of this stage of the proposal
small.

↩ [*Proposal*][proposal]

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The following small details are likely to be bikeshed prior to Stage 2 acceptance or
stabilisation and aren't pertinent to the overall design:

## What should the "compatible" value of `build-std` be named?
[unresolved-compatible]: #what-should-the-compatible-value-of-build-std-be-named

It could be named "target-modifiers" or "automatic".

↩ [*Proposal*][proposal]

## What should the interface to rustc compatibility checking be?
[unresolved-rustc-compat-interface]: #what-should-the-interface-to-rustc-compatibility-checking-be

Should it be an `--emit` flag? A `--print` flag?

↩ [*Proposal*][proposal]

# Future possibilities
[future-possibilities]: #future-possibilities

There are not currently any documented follow-ups to Stage 2.

[stage1a]: ./4-stage-1a.md#proposal
[stage1a-why-release]: ./4-stage-1a.md#why-does-always-rebuild-in-release-profile
[stage1b]: ./5-stage-1b.md#proposal