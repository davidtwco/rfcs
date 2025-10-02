- Feature Name: `build-std-compatible`
- Start Date: 2025-06-05
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

<!-- This RFC will be submitted separately from build-std RFC after the it is accepted -->

# Summary
[summary]: #summary

This RFC proposes extending the `build-std` option with a new `compatible`
value, which will become the default and automatically rebuild the standard
library when it is necessary to maintain compatibility with the compiler flags
used by the rest of the crate graph.

# Motivation
[motivation]: #motivation

This RFC aimed at unblocking the stabilisation of ABI-modifying compiler flags
as per the
[motivations described in the build-std RFC](./0000-build-std/3-motivation.md).

# Explanation
[explanation]: #explanation

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

Documentation for target modifiers will be updated to reflect that changing a
target modifier will trigger a rebuild of the standard library.

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
> standard library to ensure compatibility. Cargo will only need to run rustc
> once to determine this, comparing the flags from the user's profile against
> one of the rlibs in the pre-built standard library (assuming that they are all
> compatible with each other).

The standard library will be rebuilt in its release profile and will only vary
in the target modifier flags necessarily for it to be compatible
([?][rationale-release-profile]).

Pre-built available? | User's profile | Target modifiers changed? | Standard library re-built?
-------------------- | -------------- | ------------------------- | --------------------------
No                   | N/A            | N/A                       | Yes, std's `release` (by necessity)
Yes                  | `dev`          | Unchanged                 | No, use pre-built
Yes                  | `release`      | Unchanged                 | No, use pre-built
Yes                  | `release`      | Changed                   | Yes, std's `release`, with the addition of the target modifier flag

> [!IMPORTANT]
>
> It is assumed that enabling a target modifier for a Cargo project will happen
> by setting an options in a Cargo profiles. This RFC does not propose adding
> any specific target modifier flags to Cargo's profiles, that can happen later
> on a flag-by-flag basis, as is typical, it only assumes that this is the
> mechanism that will eventually be used. As such, there is no row in the above
> table for "target modifiers changed, profile unchanged".

As with the "always" option, the exact crates from the standard library to be
built are determined by the `build-std-crate` option or explicit dependencies on
the standard library if [*Standard library dependencies*][deps] are implemented.

Multi-target projects (resulting from multiple `--target` flags, the "target"
field in artifact dependencies or the use of `per-pkg-target` fields) results in
the decision to rebuild the standard library being made multiple times - once
for each target in the project.

As Cargo does not have per-target profiles nor a way to change the standard
library's profile on a per-target basis, the only way to configure the standard
library differently for different targets is with the use of the `[target]`
sections in the Cargo config.

The default value of all target modifier flags must be the same as their default
values as exposed in Cargo and as used with the pre-built standard library, to
avoid spurious rebuilds.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This section aims to justify all of the decisions made in the proposed design
from [*Explanation*][explanation] and discuss why alternatives were not chosen.

## Why default to "compatible"?
[rationale-default]: #why-default-to-compatible

"compatible" will only trigger when it is necessary to ensure that a user's
current configuration will build successfully. It shouldn't trigger rebuilds for
the user or change the standard library they are using if it isn't strictly
required.

If the default were not "compatible" then when the user enabled a target
modifier through Cargo (when such options are exposed), the user would
immediately face a compilation error and need to go learn about
`build-std = "compatible"` anyway.

↩ [*Explanation*][explanation]

## Why rebuild the standard library automatically?
[rationale-why-automatic]: #why-rebuild-the-standard-library-automatically

Rebuilds of the standard library happening transparently reduce the requirement
that users learn about build-std as something to enable and configure. Combined
with explicit dependencies on the standard library crates from
[*Standard library dependencies*][deps] or `build-std-crate` from
[`build-std="always"`][always], build-std can avoid any cost on users that do
not require it (by triggering automatically when a target modifier is changed,
and having no unnecessary rebuilds otherwise).

↩ [*Explanation*][explanation]

## Why add compatibility checks in rustc to support build-std?
[rationale-rustc-support]: #why-add-compatibility-checks-in-rustc-to-support-build-std

Without support from rustc, Cargo would need to assume that the pre-built
standard library's configuration is entirely configured in its Cargo profile and
would need to compare the profile of the standard library with the profile of
the user's crate and know which rustc flags are target modifiers.

Furthermore, Cargo does not need to know which flags are target modifiers with
this mechanism, it can just pass all the flags it would normally pass to rustc
(incl. from `RUSTFLAGS`) and be notified of whether a rebuild is required.

↩ [*Explanation*][explanation]

## Why build in release profile?
[rationale-release-profile]: #why-build-in-release-profile

As in [*"Why does "always" rebuild in release profile?"*][always-why-release],
building in the release profile minimises the differences between a newly-built
std and pre-built std, keeping the implications of this RFC small.

↩ [*Explanation*][explanation]

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The following small details are likely to be bikeshed prior to RFC acceptance or
stabilisation and aren't pertinent to the overall design:

## What should the "compatible" value of `build-std` be named?
[unresolved-compatible]: #what-should-the-compatible-value-of-build-std-be-named

It could be named "target-modifiers" or "automatic".

↩ [*Explanation*][explanation]

## What should the interface to rustc compatibility checking be?
[unresolved-rustc-compat-interface]: #what-should-the-interface-to-rustc-compatibility-checking-be

Should it be an `--emit` flag? A `--print` flag?

↩ [*Explanation*][explanation]

# Prior art
[prior-art]: #prior-art

See the [*Background*][background] and [*History*][history] of the build-std RFC.

# Future possibilities
[future-possibilities]: #future-possibilities

There are not currently any documented follow-ups to this RFC.

[background]: ./0000-build-std/1-background.md
[history]: ./0000-build-std/2-history.md
[always]: ./0000-build-std/4-build-std-always.md#proposal
[always-why-release]: ./0000-build-std/4-build-std-always.md#why-does-always-rebuild-in-release-profile
[deps]: ./0000-build-std/5-standard-library-dependencies.md#proposal