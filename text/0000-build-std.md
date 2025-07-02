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
  "rationale-foo", not "foo).
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

# Background
[background]: #background

See [*Implementation summary*][implementation-summary] for a summary of the
current unstable build-std feature in Cargo. This section aims to introduce any
relevant details about the standard library and compiler that are assumed
knowledge by referenced sources and later sections.

## Standard library
[background-standard-library]: #standard-library

Since the first stable release of Rust, the standard library has been distributed
as a pre-built artifact via rustup, which has a variety of advantages and/or
rationale:

- It saves Rust users from having to rebuild the standard library whenever they
  start a project or do a clean build
- The standard library has and has had dependencies which require a more complicated
  build environment than typical Rust projects
  - e.g. requiring a working C toolchain to build `compiler-builtins`' `c` feature
- To varying degrees at different times in its development, the standard library's
  implementation has been tied to the compiler implementation and has had to change
  in lockstep

Not all targets support the standard library or have a pre-built standard
library distributed via rustup. This depends on the tier of support for the
target. According to rustc's [platform support][platform-support] documentation,
for tier three targets:

> Tier 3 targets are those which the Rust codebase has support for, but which
> the Rust project does not build or test automatically, so they may or may not
> work. Official builds are not available.

..and tier two targets:

> The Rust project builds official binary releases of the standard library (or,
> in some cases, only the core library) for each tier 2 target, and automated
> builds ensure that each tier 2 target can be used as build target after each
> change.

..and finally, tier one targets:

> The Rust project builds official binary releases for each tier 1 target, and
> automated testing ensures that each tier 1 target builds and passes tests
> after each change.

All of the standard library crates leverage permanently unstable features
provided by the compiler that will never be stabilised and therefore require
nightly to build.

The configuration for the pre-built standard library build is spread across
bootstrap, the standard library workspace, individual standard library crate
manifests and the target specification. The pre-built standard library is
installed into the sysroot.

At the beginning of compilation, unless the crate has the `#![no_std]`
attribute, the compiler will load the `libstd.rlib` file from the sysroot as a
dependency of the current crate and add an implicit `extern crate std` for it.
This is the mechanism by which every crate has an implicit dependency on the
standard library.

The standard library sources are distributed in the `rust-src` component by
rustup and placed in the sysroot under `lib/rustlib/src/`. The sources consist
of the `library/` workspace plus `src/llvm-project/libunwind`, which was
required in the past to build the `unwind` crate on some targets.

Cargo supports explicitly declaring a dependency on the standard library with
a `path` source (e.g. `core = { path = "../my_core" }`), but crates with these
dependencies are not accepted by crates.io. There are crates on GitHub that
use this pattern, such as [embed-rs/stm32f7-discovery][embed-rs-cargo-toml],
which are used as `git` dependencies of other crates on GitHub.

### Dependencies
[background-dependencies]: #dependencies

Behind the facade, the standard library is split into multiple crates, some of
which are in different repositories and included as submodules or using [JOSH].

As well as local crates, the standard library depends on crates from crates.io.
It needs to be able to point these crates' dependencies on the standard library
at the sources of `core`, `alloc` and `std` in the current [rust-lang/rust]
checkout.

This is achieved through use of the `rustc-dep-of-std` feature. Crates used in
the dependency graph of `std` declare a `rustc-dep-of-std` feature and when
enabled, add new dependencies on `rustc-std-workspace-{core,alloc,std}`.
`rustc-std-workspace-{core,alloc,std}` are empty crates published on crates.io.
As part of the workspace for the standard library,
`rustc-std-workspace-{core,alloc,std}` are patched with a `path` source to the
directory for the corresponding crate.

Historically, there have necessarily been C dependencies of the standard library,
increasing the complexity of the build environment required. While these have
largely been removed over time - for example, `libbacktrace` previously depended
on `backtrace-sys` but now uses `gimli` ([rust#46439]) - there are still some C
dependencies:

- `libunwind` will either link to the LLVM `libunwind` or the system's
  `libunwind`/`libgcc_s`. LLVM's `libunwind` is shipped as part of the
  rustup component for the standard library and will be linked against
  when `-Clink-self-contained` is used
  - This only applies to Linux and Fuchsia targets
- `compiler_builtins` has an optional `c` feature that will use optimised
  routines from `compiler-rt` when enabled, which is enabled for the pre-built
  standard library
- `compiler_builtins` has an optional `mem` feature that provides symbols
  for common memory routines (e.g. `memcpy`)
  - It isn't used when `std` is built as `libc` provides these routines,
    but is often used by `no_std` crates when there is not a system `libc`
- To use sanitizers, the sanitizer runtimes from LLVM's compiler-rt need to
  be linked against. Building of these is enabled in `bootstrap.toml`
  ([`build.sanitizers`][bootstrap-sanitizers]) and they are
  included in the rustup components shipped by the project.

### Features
[background-features]: #features

There are a handful of features defined in the standard library crates'
`Cargo.toml`s. There is currently no stable existing mechanism for users to
enable or disable these features. The default set of features is determined by
[logic in bootstrap][bootstrap-features-logic] and [the `rust.std-features`
key in `bootstrap.toml`][bootstrap-features-toml]. The enabled features are
often different depending on the target.

### Target support
[background-target-support]: #target-support

The `std` crate's [`build.rs`][std-build.rs] checks for supported values of the
`CARGO_CFG_TARGET_*` environment variables. These variables are akin to the
conditional compilation [configuration options][conditional-compilation-config-options],
and often correspond to parts of the target triple (for example,
`CARGO_CFG_TARGET_OS` corresponds to the "os" part of a target triple - "linux"
in "aarch64-unknown-linux-gnu"). This filtering is strict enough to distinguish
between built-in targets but loose enough to match similar custom targets.

When encountering an unknown or unsupported operating system then the
`restricted_std` cfg is set. `restricted_std` marks the entire standard library
as unstable, requiring `feature(restricted_std)` to be enabled on any crate that
depends on it. There is no mechanism for users to enable the `restricted_std`
feature on behalf of dependencies. There is also no such mechanism for `alloc`
or `core`, only `std`.

Cargo and rustc support custom targets, defined in JSON files according to an
unstable schema defined in the compiler. On nightly, users can dump the
target-spec-json for an existing target using `--print target-spec-json`. This
JSON can be saved in a file, tweaked and used as the argument to `--target` even
on stable toolchains, though the Rust project does not officially support them
and the JSON format is itself unstable. Custom targets do not have a pre-built
standard library and so must use `-Zbuild-std`. Custom targets may have
`restricted_std` set depending on their `cfg` configuration options - generally
speaking depending on how similar they are to builtin targets.

## Registries
[background-registries]: #registries

Cargo's building of the dependency graph is driven by the registry index.
[Cargo registries][cargo-docs-registry], like crates.io, are centralised sources
for crates. A registry's index is the interface between Cargo and the registry
that Cargo queries to know which crates are available, what their dependencies
are, etc. crates.io's registry index is a Git repository -
[rust-lang/crates.io-index] - which is updated automatically by crates.io when
crates are published, yanked, etc. Cargo can query registries using a Git
protocol which caches the registry on disk, or using a sparse protocol which
exposes the index over HTTP and allows Cargo to avoid Cargo having a local copy
of the whole index, which has become quite large for crates.io.

Each crates in the registry has a JSON file, following
[a defined schema][cargo-json-schema]. Crates may refer to those in other
registries, but all crates in the dependency graph must exist in a registry. As
the registry index drives the building of Cargo's dependency graph, all crates
that end up in the dependency graph must be present a registry.

Registries can have different policies for what crates are accepted. For
example, crates.io does not permit publishing packages named `std` or `core` but
other registries might.

## Panic strategies
[background-panic-strategies]: #panic-strategies

Rust has the concept of a *panic handler*, which is a crate that is responsible
for performing a panic. There are various panic handler crates on crates.io,
such as [panic-abort] (which different from the `panic_abort` panic runtime!),
[panic-halt], [panic-itm], and [panic-semihosting]. Panic handler crates define
a function annotated with `#[panic_handler]`. There can only be one
`#[panic_handler]` in the crate graph.

`core` uses the panic handler to implement panics inserted by code generation
(e.g. arithmetic overflow or out-of-bounds access) and the `core::panic!` macro
immediately delegates to the panic handler crate.

`std` is also a panic handler. `std`'s panic handler and `std::panic!` macro
print panic information to stderr and delegate to a *panic runtime* to decide
what to do next, determined by the *panic strategy*.

There are two panic runtime crates in the standard library - `panic_unwind` and
`panic_abort` - each with a corresponding panic strategy. Each target supported
by rustc specifies a default panic strategy - either "unwind" or "abort" -
though these are only relevant if `std`'s panic handler is used (i.e. the target
isn't a `no_std` target or being used with a `no_std` crate).

Rust's `-Cpanic` flag allows the user to choose the panic strategy, with the
target's default as a fallback. If `-Cpanic=unwind` is provided then this
doesn't guarantee that the unwind strategy is used, as the target may not
support it.

Both crates are compiled and shipped with the pre-built standard library for
targets which support `std`. Some targets have a pre-built standard library with
only the `core` and `alloc` crates, such as the `x86_64-unknown-none` target.
While `x86_64-unknown-none` defaults to the `abort` panic strategy, as this
target does not support the standard library, this default isn't actually
relevant.

The `std` crate has a `panic_unwind` feature that enables an optional dependency
on the `panic_unwind` crate.

`core` also has a `panic_immediate_abort` feature which modifies the
`core::panic!` macro to immediately call the abort intrinsic without calling the
panic handler. `std` and `alloc` have the same feature which enable the feature
in `core`. `std`'s feature also adds an immediate abort to its `panic!` macro.

## Target modifiers
[background-target-modifiers]: #target-modifiers

[rfcs#3716] introduced the concept of *target modifiers* to rustc. Flags marked
as target modifiers must match across the entire crate graph or the compilation
will fail.

For example, flags are made target modifiers when they change the ABI of
generated code and could result in unsound ABI mismatches if two crates are
linked together with different values of the flag set.

# History
[history]: #history

*The following summary of the prior art is necessarily less detailed than the
source material, which is exhaustively surveyed in
[Appendix II: Exhaustive literature review][appendix-ii].*

## [rfcs#1133] (2015)
[rfcs-1133-2015]: #rfcs1133-2015

build-std was first proposed in a [2015 RFC (rfcs#1133))][rfcs#1133] by
[Ericson2314], aiming to improve support for targets that do not have a
pre-built standard library; to enable building the standard library with
different profiles; and to simplify `rustbuild` (now `bootstrap`). It also was
written with the goal of supporting the user in providing a custom
implementation of the standard library and supporting different implementations
of the language that provide their own standard libraries.

This RFC proposed that the standard library be made an explicit dependency in
`Cargo.toml` and be rebuilt automatically when required. An implicit dependency
on the standard library would be added automatically unless an explicit
dependency is written. This RFC was written prior to a stable `#![no_std]`
attribute and so does not address the circumstance where a implicit dependency
would make a no-std crate fail to compile on a target that does not support
the standard library.

There were objectives of and possibilities enabled by the RFC that were not
shared with the project teams at the time, such as the standard library being
a regular crate on crates.io and the concept of the sysroot being retired.
Despite this, the RFC appeared to be close to acceptance before being blocked
by Cargo having a mechanism to have unstable features and then closed in favour
of [cargo#4959].

## [xargo] and [cargo#4959] (2016)
[xargo-and-cargo-4959-2016]: #xargo-and-cargo4959-2016

While the discussions around [rfcs#1133] where ongoing, [xargo] was released in
2016. Xargo is a Cargo wrapper that builds a sysroot with a customised standard
library and then uses that with regular Cargo operations (i.e. `xargo build`
performs the same operation as `cargo build` but with a customised standard
library). Configuration for the customised standard library was configured in
the `Xargo.toml`, supporting configuring codegen flags, profile settings, Cargo
features and multi-stage builds. It required nightly to build the standard
library as it did not use `RUSTC_BOOTSTRAP`. Xargo had inherent limitations due
to being a Cargo wrapper, leading to suggestions that its functionality be
integrated into Cargo.

[cargo#4959] is a proposal inspired by [xargo], suggesting that a `[sysroot]`
section be added to `.cargo/config` which would enable similar configuration to
that of `Xargo.toml`. If this configuration is set, Cargo would build and use a
sysroot with a customised standard library according to the configuration
specified and the release profile. This sysroot would be rebuilt whenever
relevant configuration changes (e.g. profiles). [cargo#4959] received varied
feedback: the proposed syntax was not sufficiently user-friendly; it did not
enable the user to customise the standard library implementation; and that
exposing bootstrap stages was brittle and user-unfriendly. [cargo#4959] wasn't
updated after submission so ultimately stalled and remains open.

[rfcs#1133] and [cargo#4959] took very different approaches to build-std, with
[cargo#4959] proposing a simpler approach that exposed the necessary low-level
machinery to users and [rfcs#1133] attempting to take a more first-class and
user-friendly approach that has many tricky design implications.

## [rfcs#2663] (2019)
[rfcs-2663-2019]: #rfcs2663-2019

In 2019, [*rfcs#2663: `std` Aware Cargo*][rfcs#2663] was opened as the most
recent RFC attempting to advance build-std. [rfcs#2663] shared many of the
motivations of [rfcs#1133]: building the standard library for tier three and
custom targets; customising the standard library with different Cargo features;
and applying different codegen flags to the standard library. It did not concern
itself with build-std's potential use in `rustbuild` or with abolishing the
sysroot.

[rfcs#2663] was primarily concerned what functionality should be available to
the user and what the user experience ought to be. It proposed that `core`,
`alloc` and `std` be automatically built when the target did not have a pre-built
standard library available through rustup. It would be automatically rebuilt on
any target when the profile configuration was modified such that it no longer
matched the pre-built standard library. If using nightly, the user could enable
Cargo features and modify the source of the standard library. Standard library
dependencies were implicit by default, as today, but would be written explicitly
when enabling Cargo features. It also aimed to stabilise the target-spec-json
format and allow "stable" Cargo features to be enabled on stable toolchains, and
as such proposed the concept of stable and unstable Cargo features be
introduced.

There was a lot of feedback on [rfcs#2663] which largely stemmed from it being
very high-level, containing many large unresolved questions and details left for
the implementors to work out. For example, it proposed that there be a concept
of stable and unstable Cargo features but did not elaborate any further, leaving
that as an implementation detail. Nevertheless, the proposal was valuable in
more clearly elucidating a potential user experience that build-std could aim
for, and the feedback provided was incorporated into the [wg-cargo-std-aware]
effort, described below.

## [wg-cargo-std-aware] (2019-)
[wg-cargo-std-aware-2019-]: #wg-cargo-std-aware-2019-

[rfcs#2663] demonstrated that there was demand for a mechanism for being able to
(re-)build the standard library, and the feedback showed that this was a thorny
problem with lots of complexity, so in 2019, the [wg-cargo-std-aware] repository
was created to organise related work and explore the issues involved in
build-std.

[wg-cargo-std-aware] led to the current unstable implementation of `-Zbuild-std`
in Cargo, which is described in detail in the [*Implementation summary*
section][implementation-summary] below.

Issues in the wg-cargo-std-aware repository can be roughly partitioned into seven
categories:

1. **Exploring the motivations and use cases for the standard library**

   There are a handful of motivations catalogued in the [wg-cargo-std-aware]
   repository, corresponding to those raised in the earlier RFCs and proposals:

   - Building with custom profile settings ([wg-cargo-std-aware#2])
   - Building for unsupported targets ([wg-cargo-std-aware#3])
   - Building with different Cargo features ([wg-cargo-std-aware#4])
   - Replacing the source of the standard library ([wg-cargo-std-aware#7])
   - Using build-std in bootstrap/rustbuild ([wg-cargo-std-aware#19])
   - Improving the user experience for `no_std` binary projects
     ([wg-cargo-std-aware#36])

   These are all either fairly self-explanatory, described in the summary of the
   previous RFCs/proposals above, or in the [*Motivation*][motivation] section
   of this RFC.

2. **Support for build-std in Cargo's subcommands**

   Cargo has various subcommands where the desired behaviour when used with
   build-std needs some thought and consideration. A handful of issues were
   created to track this, most receiving little to no discussion:
   [`cargo metadata`][wg-cargo-std-aware#20], [`cargo clean`][wg-cargo-std-aware#21],
   [`cargo pkgid`][wg-cargo-std-aware#24], and [the `-p` flag][wg-cargo-std-aware#26].

   [`cargo fetch`][wg-cargo-std-aware#22] had fairly intuitive interactions with
   build-std - that `cargo fetch` should also fetch any dependencies of the
   standard library - which was implemented in [cargo#10129].

   The [`--build-plan` flag][wg-cargo-std-aware#45] does not support build-std and its
   issue did not receive much discussion, but the future of this flag in its
   entirety seems to be uncertain.

   [`cargo vendor`][wg-cargo-std-aware#23] did receive lots of discussion.
   Vendoring  the standard library is desirable (for the same reasons as any
   vendoring), but would lock the user to a specific version of the toolchain
   when using a vendored standard library. However, if the `rust-src` component
   contained already-vendored dependencies, then `cargo vendor` would not need
   to support build-std and users would see the same advantages.

   Vendored standard library dependencies were implemented using a hacky
   approach (necessarily, prior to the standard library having its own
   workspace), but this was later reverted due to bugs. No attempt has been made
   to reimplement vendoring since the standard library has had its own
   workspace.

3. **Dependencies of the standard library**

   There are a handful of dependencies of the standard library that may pose
   challenges for build-std by dint of needing a working C toolchain or
   special-casing.

   [`libbacktrace`][wg-cargo-std-aware#16] previously required a C compiler to
   build `backtrace-sys`, but now uses `gimli` internally.

   [`compiler_builtins`][wg-cargo-std-aware#15] has a `c` feature that uses C
   versions of some intrinsics that are more optimised. This is used by the
   pre-built standard library, and if not used by build-std, could be a point of
   divergence. `compiler-builtins/c` can have a significant impact on code
   quality and build size. It also has a `mem` feature which provides symbols
   (`memcopy`, etc) for platforms without `std` that don't have these same
   symbols provided by `libc`. compiler-builtins is also built with a large
   number of compilation units to force each function into a different unit.

   [Sanitizers][wg-cargo-std-aware#17], when enabled, require a sanitizer
   runtime to be present. These are currently built by bootstrap and part of
   LLVM.

4. **Design considerations**

   There are many design considerations discussed in the [wg-cargo-std-aware]
   repository:

   [wg-cargo-std-aware#5] explored how/if dependencies on the standard library
   should be declared. The issue claims that users should have to opt-in to
   build-std, support alternative standard library implementations, and that
   Cargo needs to be able to pass `--extern` to rustc for all dependencies.

   It is an open question how to handle multiple dependencies each declaring a
   dependency on the standard library. A preference towards unifying standard
   library dependencies was expressed (these would have no concept of a version,
   so just union all features).

   There was no consensus on how to find a balance between explicitly depending
   on the standard library versus implicitly, or on whether the pre-built-ness
   of a dependency should be surfaced to the user.

   [wg-cargo-std-aware#6] argues that target-spec-json would be de-facto stable
   if it can be used by build-std on stable. While `--target=custom.json` can be
   used on stable today, it effectively requires build-std and so a nightly
   toolchain. As build-std enables custom targets to be used on stable, this
   would effectively be a greater commitment to the current stability of custom
   targets than currently exists and would warrant an explicit decision.

   [wg-cargo-std-aware#8] highlighted that a more-portable standard library
   would be beneficial for build-std (i.e. a `std` that could build on any
   target), but that making the standard library more portable isn't necessarily
   in-scope for build-std.

   [wg-cargo-std-aware#11] investigated how build-std could get the standard
   library sources. rustup can download `rust-src`, but there was a preference
   expressed that rustup not be required. Cargo could have reasonable default
   probing locations that could be used by distros and would include where
   rustup puts `rust-src`.

   [wg-cargo-std-aware#12] concluded that the `Cargo.lock` of the standard
   library would need to be respected so that the project can guarantee that the
   standard library works with the project's current testing.

   [wg-cargo-std-aware#13] aimed to determine how to determine the default set
   of cfg values for the standard library. This is currently determined by
   bootstrap. This could be duplicated in Cargo in the short-term, made visible
   to build-std through some configuration, or require the user to explicitly
   declare them.

   [wg-cargo-std-aware#14] looks into additional rustc flags and environment
   variables passed by bootstrap to the compiler. A comparison of the
   compilation flags from bootstrap and build-std was [posted in a comment][wg-cargo-std-aware#14-review].
   No solutions were suggested, other than that it may need a similar mechanism
   as [wg-cargo-std-aware#13].

   [wg-cargo-std-aware#29] tries to determine how to support different panic
   strategies. Should Cargo use the profile to decide what to use? How does it
   know which panic strategy crate to use? It is argued that Cargo ought to work
   transparently - if the user sets the panic strategy differently then a
   rebuild is triggered.

   [wg-cargo-std-aware#30] identifies that some targets have special handling in
   bootstrap which will need to be duplicated in build-std. Targets could be
   whitelisted or blacklisted to avoid having to address this initially.

   [wg-cargo-std-aware#38] argues that a forced lock of the standard library
   is desirable, to which there was no disagreement. This was more relevant
   when build-std did not use the on-disk `Cargo.lock`.

   [wg-cargo-std-aware#39] explores the interaction between build-std and
   public/private dependencies ([rfcs#3516]). Should the standard library always
   be public? There were no solutions presented, only that if defined in
   `Cargo.toml`, the standard library will likely inherit the default from that.

   [wg-cargo-std-aware#43] investigates the options for the UX of build-std.
   `-Zbuild-std` flag is not a good experience as it needs added to every
   invocation and has few extension points. Using build-std should be a unstable
   feature at first. It was argued that build-std should be transparent and
   happen automatically when Cargo determines it is necessary. There are
   concerns that this could trigger too often and that it should only happen
   automatically for ABI-modifying flags.

   [wg-cargo-std-aware#46] observes that some targets link against special
   object flags (e.g. `crt1.o` on musl) and that build-std will need to handle
   these without hardcoding target-specific logic. There were no conclusions,
   but `-Clink-self-contained` might be able to help.

   [wg-cargo-std-aware#47] discusses how to handle targets that typically ship
   with a different linker (e.g. `rust-lld` or `gcc`). `rust-lld` is now shipped
   by default reducing the potential impact of this, though it is discovered via
   the sysroot, and so will need to be found via another mechanism if disabled.

   [wg-cargo-std-aware#50] argues that the impact on build probes ought to be
   considered and was later closed as t-cargo do not want to support build
   probes.

   [wg-cargo-std-aware#51] plans for removal of `rustc-dep-of-std`, identifying
   that if explicit dependencies on the standard library are adopted, that the
   need for this feature could be made redundant.

   [wg-cargo-std-aware#68] notices that `profiler_builtins` needs to be compiled
   after `core` (i.e. `core` can't be compiled with profiling). The error
   message has been improved for this but there was otherwise no commentary.
   This has changed since the issue was filed, as `profiler_builtins` is now a
   `#![no_core]` crate.

   [wg-cargo-std-aware#85] considers that there has to be a deliberate testing
   strategy in place between the [rust-lang/rust] and [rust-lang/cargo]
   repositories to ensure there is no breakage. `rust-toolstate` could be used
   but is not very good. Alternatively, Cargo could become a [JOSH] subtree of
   [rust-lang/rust].

   [wg-cargo-std-aware#86] proposes that the initial set of targets supported by
   build-std be limited at first to further reduce scope and limit exposure to
   the trickier issues.

   [wg-cargo-std-aware#88] reports that `cargo doc -Zbuild-std` doesn't generate
   links to the standard library. Cargo doesn't think the standard library comes
   from crates.io, and bootstrap isn't involved to pass
   `-Zcrate-attr="doc(html_root_url=..)"` like in the pre-built standard
   library.

   [wg-cargo-std-aware#90] asks how `restricted_std` should apply to custom
   targets. `restricted_std` is triggered based on the `target_os` value, which
   means it will apply for some custom targets but not others. build-std needs
   to determine what guarantees are desirable/expected. Current implementation
   wants slightly-modified-from-default target specs to be accepted and
   completely new target specs to hit `restricted_std`.

   [wg-cargo-std-aware#92] suggests that some targets could be made "unstable"
   and as such only support build-std on nightly. This forces users of those
   targets to use nightly where they will receive more frequent fixes for their
   target. It would also permit more experimentation with build-std while
   enabling stabilisation for mainstream targets.

5. **Implementation considerations**
   These won't be discussed in this summary, see [the implementation summary][implementation-summary]
   or [the relevant section of the literature review for more detail][implementation]

6. **Bugs in the compiler or standard library**
   These aren't especially relevant to this summary, see [the relevant section
   of the literature review for more detail][bugs-in-the-compiler-or-standard-library]

7. **Cargo feature requests narrowly applied to build-std**
   These aren't especially relevant to this summary, see [the relevant section
   of the literature review for more detail][cargo-feature-requests-narrowly-applied-to-build-std]

Since around 2020, activity in the [wg-cargo-std-aware] repository largely
trailed of and there have not been any significant developments related to
build-std since.

### Implementation summary
[implementation-summary]: #implementation-summary

*An exhaustive review of implementation-related issues, pull requests and
discussions can be found in [the relevant section of the literature review][implementation].*

There has been an unstable and experimental implementation of build-std in Cargo
since August 2019 ([wg-cargo-std-aware#10]/[cargo#7216]).

[cargo#7216] added the [`-Zbuild-std`][build-std] flag to Cargo. `-Zbuild-std`
re-builds the standard library crates which rustc then uses instead of the
pre-built standard library from the sysroot.

`-Zbuild-std` builds `std` by default. `test` is also built if tests are being
run. Optionally, users can provide the list of crates to be built, though this
was intended as an escape hatch to work around bugs - the arguments to the flag
are semi-unstable since the names of crates comprising the standard
library are not stable.

Cargo has a hardcoded list of what dependencies need to be added for a given
user-requested crate (i.e. `std` implies building `core`, `alloc`,
`compiler_builtins`, etc.). It is common for users to manually specify the
`panic_abort` crate.

Originally, `-Zbuild-std` required that `--target` be provided
([wg-cargo-std-aware#25]) to force Cargo to use different sysroots for the host
and target , but this restriction was later resolved ([cargo#14317]).

A second flag, [`-Zbuild-std-features`][build-std-features], was added in
[cargo#8490] and allows overriding the default Cargo features of the standard
library. Like the arguments to `-Zbuild-std`, this values accepted by this flag
are inherently unstable as the library team has not committed to any of the
standard library's Cargo features being stable. Features are enabled on the
`sysroot` crate and propagate down through the crate graph of the standard
library (e.g. `compiler-builtins-mem` is a feature in `sysroot`, `std`,
`alloc`, and `core` until `compiler_builtins`).

build-std gets the source of the standard library from the `rust-src` rustup
component. This does not happen automatically and the user must ensure the
component has been downloaded themselves. Only the standard library crates from
the [rust-lang/rust] repository are included in the `rust-src` depdendency (i.e.
none of the crates.io dependencies).

When `-Zbuild-std` has been passed, Cargo creates a second workspace for the
standard library based on the `Cargo.{toml,lock}` from the `rust-src` component.
Originally this was a virtual workspace, prior to the standard library having a
separate workspace from the compiler which could be used independently
([rust#128534]/[cargo#14358]). This workspace is then resolved separately and
the resolve is combined with the user's resolve to produce a dependency graph of
things to build with the user's crates depending on the standard library's
crates. Some additional work is done to deduplicate crates across the graph and
then this crate graph is used to drive work (usually rustc invocations) as
usual. This approach allows for build-time parallelism and sharing of crates
between the two separate resolves but does involve `build-std`-specific logic in
and around unit generation and is very unlike the rest of Cargo
([wg-cargo-std-aware#64]).

Resolving the standard library separately from the user's crate helps guarantee
that the exact dependency versions of the pre-built standard library are used,
which is a key constraint ([wg-cargo-std-aware#12]). Locking the standard
library could also help ([wg-cargo-std-aware#38]). A consequence of this is that
each of the Cargo subcommands (e.g. `cargo metadata`) need to have special
support for build-std implemented, but this might be desirable.

The standard library crates are considered non-local packages and so are not
compiled with incremental compilation or dep-info fingerprint tracking and any
warnings will be silenced.

build-std provides newly-built standard library dependencies to rustc using
`--extern noprelude:$crate`. `noprelude` was added in [rust#67074] to support
build-std and ensure that loading from the sysroot and using `--extern` were
equivalent ([wg-cargo-std-aware#40]). Prior to the addition of `noprelude`,
build-std briefly created new sysroots and used those instead of `--extern`
([cargo#7421]). rustc can still try to load a crate from the sysroot if the user
uses it which is currently a common source of confusing "duplicate lang item"
errors (as the user ends up with build-std `core` and sysroot `core`
conflicting).

Host dependencies like build scripts and `proc_macro` crates use the
existing pre-built standard library from the sysroot, so Cargo does not
pass `--extern` to those.

Modifications to the standard library are not supported. While build-std
has no mechanism to detect or prevent modifications to the `rust-src` content,
rebuilds aren't triggered automatically on modifications. The user cannot
override dependencies in the standard library workspace with `[patch]` sections
of their `Cargo.toml`.

To simplify build-std in Cargo, build-std wants to be able to always build
`std`, which is accomplished through use of the
[`unsupported` module in `std`'s platform abstraction layer][std-unsupported],
and `restricted_std`. `std` checks for unsupported targets in its
[`build.rs`][std-build.rs] and applies the `restricted_std` cfg which marks the
standard library as unstable for unsupported targets.

Users can enable the `restricted_std` feature in their crates. This mechanism
has been noted as confusing ([wg-cargo-std-aware#87]) and has the issue that the
user cannot opt into the feature on behalf of dependencies
([wg-cargo-std-aware#69]).

The initial implementation does not include support for build-std in many of
Cargo's subcommands including `metadata`, `clean`, `vendor`, `pkgid` and the
`-p` options for various commands. Support for `cargo fetch` was implemented in
[cargo#10129].

## Related work
[related-work]: #related-work

There are a variety of ongoing efforts, ideas, RFCs or draft notes describing
features that are related or would be beneficial for build-std:

- **[Opaque dependencies]**, [epage], May 2025
  - Introduces the concept of an opaque dependency that has its own
    `Cargo.lock`, `RUSTFLAGS` and `profile`
  - Opaque dependencies could enable a variety of build-time performance
      improvements:
    - Caching - differences in dependency versions can cause unique instances of
      every dependent crate
    - Pre-built binaries - can leverage a pre-built artifact for a given opaque
      dependency
      - e.g. the standard library's distributed `rlib`s
    - MIR-only/cross-crate lazy compilation - Small dependencies could be built
      lazily and larger dependencies built once
    - Optimising dependencies - dependencies could always be optimised when they
      are unlikely to be needed during debugging

# Motivation
[motivation]: #motivation

While the pre-built standard library has been sufficient for the majority of
Rust users, there are a variety of use-cases which require the ability to
re-build the standard library.

This RFC aims to support the following use cases:

1. **Re-building the standard library with different codegen flags or profile**
   ([wg-cargo-std-aware#2])

 - Embedded users need to optimise aggressively for size, due to the limited
   space available on their target platforms, which can be achieved in Cargo by
   setting `opt-level = s/z` and `panic = "abort"` in their profile. However,
   these settings will not apply to the pre-built standard library
 - Similarly, when deploying to known environments, use of `target-cpu` or
   `target-feature` can improve the performance of code generation or allow the
   use of newer hardware features than the target's baseline provides. As above,
   these configuration will not apply to the pre-built standard library
 - While the pre-built standard library is built to support debugging without
   compromising size and performance by setting `debuginfo=1`, this isn't
   ideal, and building the standard library with the dev profile would provide
   a better experience

2. **Unblock stabilisation of ABI-modifying compiler flags**

  - Any compiler flags which change the ABI cannot currently be stabilised as they
    would immediately mismatch with the pre-built standard library
    - Without an ability to rebuild the standard library using these flags, it is
      impossible to use them effectively and safely if stabilised
  - ABI-modifying flags are designated as target modifiers ([rfcs#3716]/[rust#136966])
    and require that the same value for the flag is passed to all compilation units
    - Flags which need to be set across the entire crate graph to uphold some
      property (i.e. enhanced security) are also target modifiers
    - For example: sanitizers, control flow integriy, `-Zfixed-x18`, etc

3. **Building the standard library on a stable toolchain without Cargo**

  - While tangential to the core of build-std as a feature, projects like Rust
    for Linux want to be able to build an unmodified `core` from `rust-src` in
    the sysroot on a stable toolchain without Cargo
  - Cargo may also want a mechanism to build the standard library for build-std
    on a stable toolchain without relying on `RUSTC_BOOTSTRAP`

4. **Building standard library crates that are not shipped for a target**

  - Targets which have limited `std` support may wish to use the subsets of the
    standard library which do work

5. **Using the standard library with tier three targets**

  - There is no stable mechanism for using the standard library on a tier three
    target that does not ship a pre-built std
  - While it is common for these targets to not support the standard library,
    they should be able to use `core`
  - These users are forced to use nightly and the unstable `-Zbuild-std`
    feature or third-party tools like [cargo-xbuild] (formerly [xargo])

6. **Using miri on a stable toolchain**

  - Using miri requires building the standard library with specific compiler flags
    that would not be appropriate for the pre-built standard library, so is forced
    to require nightly and build its own sysroot

The following use cases are not supported by this RFC, but could be supported
with follow-up RFCs (and this RFC will attempt to ensure they remain viable as
future possiblities):

1. **Using the standard library with custom targets**

  - There is no stable mechanism for using the standard library for a custom
    target (using target-spec-json)
  - Like tier three targets, these targets often only support `core` and are
    forced to use nightly today

2. **Enabling Cargo features for the standard library** ([wg-cargo-std-aware#4])

  - There are opportunities to expose Cargo features from the standard library that
    would be useful for certain subsets of the Rust users.
    - For example, embedded users may want to enable a feature like `optimize_for_size` or
      `panic_immediate_abort` to reduce binary size

Some use cases are unlikely to supported by the project unless a new and
compelling use-case is presented, andsso this RFC may make decisions which make
these motivations harder to solve in future:

1. **Modifying the source code of the standard library** ([wg-cargo-std-aware#7])

  - Some platforms require a heavily modified standard library that would not
    be suitable for upstreaming, such as [Apache's SGX SDK][sgx] which replaces
    some standard library and ecosystem crates with forks or custom crates for a
    custom `x86_64-unknown-linux-sgx` target
  - Similarly, some tier three targets may wish to patch standard library
    dependencies to add or improve support for the target

# Detailed explanation
[detailed-explanation]: #detailed-explanation

This section describes the various changes proposed by this RFC that build-std
would be comprised of.

## Target standard library support
[target-standard-library-support]: #target-standard-library-support

A new `standard_library_support` field is added to the target specification
([?][rationale-target-spec-purpose]), replacing the existing `metadata.std`,
which has three fields: `core`, `alloc` and `std`
([?][rationale-target-spec-core-alloc-std]). These fields determine whether the
corresponding crate is supported for that target. On a stable toolchain,
build-std will emit an error if it required to build a crate which is not
supported by a given target.

Each target will set `core`, `alloc` and `std` as appropriate. For example, all
three standard library crates will be stable on "aarch64-unknown-linux-gnu",
only `alloc` and `core` will be stable on "x86_64-unknown-none" and only `core`
will be stable on "mipsel-sony-psx".

The `target-standard-library-support` option will be supported by rustc's
`--print` flag:

```shell-session
$ rustc --print target-standard-library-support
target: aarch64-unknown-linux-gnu
std: true
alloc: true
core: true
$ rustc --print target-standard-library-support --target x86_64-unknown-none
target: x86_64-unknown-none
std: false
alloc: true
core: true
$ rustc --print target-standard-library-support --target mipsel-sony-psx
target: mipsel-sony-psx
std: false
alloc: false
core: true
```

On a stable toolchain, if Cargo needs to build one of the standard library
crates for a target, it will check `--print target-standard-library-support` to
determine whether to emit an error.

The existing `restricted_std` mechanism will be removed from the standard
library's [`build.rs`][std-build.rs] as it is replaced by this mechanism
([?][rationale-replace-restricted-std]).

*See the following sections for rationale/alternatives:*

- [*Should target specifications own knowledge of which standard library crates are supported?*][rationale-target-spec-purpose]
- [*Why record support for `core`, `alloc` and `std` separately?*][rationale-target-spec-core-alloc-std]
- [*Why replace `restricted_std` with explicit standard library support for a target?*][rationale-replace-restricted-std]

### Custom targets
[custom-targets]: #custom-targets

Cargo will detect when the standard library is to be built for a custom target
and will emit an error ([?][rationale-disallow-custom-targets]).

> [!NOTE]
>
> Cargo could detect use of a custom target either by comparing it with the list
> of built-in targets that rustc reports knowing about (via `--print target-list`)
> or by checking if a file exists at the path matching the provided target name.

Custom targets can still be used with build-std on nightly toolchains provided
that `-Zunstable-options` is provided to Cargo.

*See the following sections for rationale/alternatives:*

- [*Why disallow custom targets?*][rationale-disallow-custom-targets]

*See the following sections for future possibilities:*

- [*Allow custom targets with build-std*][future-custom-targets]

## Standard library dependencies
[standard-library-dependencies]: #standard-library-dependencies

Users can now optionally declare explicit dependencies on the standard library
in their `Cargo.toml` files ([?][rationale-why-explicit-deps]):

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { builtin = true }
```

`builtin` is a new source of dependency, like registry dependencies (with the
`version` key and optionally the `registry` key), `path` dependencies or `git`
dependencies. `builtin` can only be set to `true` and cannot be combined with
any other dependency source for a given dependency
([?][rationale-builtin-other-sources]). `builtin` can only be used with crates
named `core`, `alloc` or `std` ([?][rationale-no-builtin-other-crates]).

An explicit dependency on a `builtin = true` crate implies a direct dependency
on other `builtin` crates that the crate depends on
([?][rationale-builtin-implied-direct]). For example, an explicit dependency on
`alloc` like so..

```toml
[dependencies]
alloc = { builtin = true }
```

..is equivalent to an explicit dependency on both `alloc` and `core`:

```toml
[dependencies]
alloc = { builtin = true }
core = { builtin = true }
```

Similarly, an explicit `std` dependency implies both `alloc` and `core`.

crates.io will accept crates published which have `builtin` dependencies.

Crates without an explicit dependency on the standard library now have a
implicit dependency on the `std` crate ([?][rationale-no-migration]). In the
`hello_world` crate below, there is an implicit dependency on `std`..

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
```

..which is equivalent to the following explicit dependency on `builtin` crates:

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { builtin = true }
alloc = { builtin = true }
core = { builtin = true }
```

Any `builtin` dependency present in the manifest will disable the implicit
dependency on `std`. Multiple standard library crates can be added explicitly as
dependencies:

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { builtin = true }
core = { builtin = true }
```

Standard library dependencies can be marked as `optional` and be enabled
conditionally by a feature in the crate:

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { builtin = true, optional = true }
core = { builtin = true }

[features]
default = ["std"]
std = ["dep:std"]
```

If there is an optional dependency on the standard library then there must be at
least one non-optional dependency on the standard library (e.g. an optional
`std` and non-optional `core` or `alloc`, or an optional `alloc` and
non-optional `core`). `core` cannot be optional.

Dependencies with `builtin = true` cannot be renamed with the `package` key
([?][rationale-package-key]).

Dependencies with `builtin = true` can be specified as platform-specific
dependencies:

```toml
[target.'cfg(unix)'.dependencies]
std = { builtin = true}
```

Implicit and explicit standard library dependencies are added to `Cargo.lock`
files ([?][rationale-cargo-lock]). 

> [!NOTE]
> 
> A version of the `Cargo.lock` file will be introduced to add support for
> packages with a `builtin` source:
>
> ```toml
> [[package]]
> name = "std"
> version = "0.0.0"
> source = "builtin"
> ```
> 
> The package version of `std`, `alloc` and `core` will be fixed at `0.0.0`. The
> optional lockfile fields `dependencies` and `checksum` will not be present for
> `builtin` dependencies.

*See the following sections for rationale/alternatives:*

- [*Why explicitly declare dependencies on the standard library in `Cargo.toml`?*][rationale-why-explicit-deps]
- [*Why disallow builtin dependencies to be combined with other sources?*][rationale-builtin-other-sources]
- [*Why imply direct builtin dependencies?*][rationale-builtin-implied-direct]
- [*Why disallow builtin dependencies on other crates?*][rationale-no-builtin-other-crates]
- [*Why not migrate to always requiring explicit standard library dependencies?*][rationale-no-migration]
- [*Why add standard library dependencies to Cargo.lock?*][rationale-cargo-lock]

*See the following sections for relevant unresolved questions:*

- [*What syntax is used to identify dependencies on the standard library in `Cargo.toml`?*][unresolved-dep-syntax]
- [*What is the format for builtin dependencies in `Cargo.lock`?*][unresolved-lockfile]

### Non-`builtin` standard library dependencies
[non-builtin-standard-library-dependencies]: #non-builtin-standard-library-dependencies

Cargo already supports `path` and `git` dependencies for crates named `core`,
`alloc` and `std` which continue to be supported and work:

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { path = "../my_std" } # already supported by Cargo
```

A `core`/`alloc`/`std` dependency with a `path`/`git` source can be combined
with `builtin` dependencies:

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { path = "../my_std" }
core = { builtin = true }
```

As before, crates with `path`/`git` dependencies for `core`, `alloc` or `std`
are not accepted by crates.io.

### Patches
[patches]: #patches

On nightly toolchains, it is permitted to patch the standard library
dependencies with `path` and `git` sources (or any other source)
([?][rationale-patching]):

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { builtin = true }

[patch.builtin] # permitted on nightly
std = { .. }

[patch.builtin] # permitted on nightly
std = { path = "../libstd" }
```

As with any other `path` or `git` dependency, crates with these dependency
sources will not be able to be published to crates.io.

It is not possible to perform source replacement on standard library
dependencies using `builtin = true`.

*See the following sections for rationale/alternatives:*

- [*Why permit patching of the standard library dependencies on nightly?*][rationale-patching]

*See the following sections for relevant unresolved questions:*

- [*What syntax is used to patch dependencies on the standard library in `Cargo.toml`?*][unresolved-patch-syntax]

### Features
[features]: #features-1

On a stable toolchain, it is not permitted to enable or disable features of
explicit standard library dependencies ([?][rationale-features]), as in the
below example:

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { builtin = true, features = [ "foo" ] } # not permitted
# ..or..
std = { builtin = true, default-features = false } # not permitted
```

*See the following sections for rationale/alternatives:*

- [*Why limit enabling standard library features to nightly?*][why-limit-enabling-standard-library-features-to-nightly]

*See the following sections for future possibilities:*

- [*Allow enabling/disabling features with build-std*][future-features]

### Public and private dependencies
[public-and-private-dependencies]: #public-and-private-dependencies

Implicit dependencies on the standard library default to being public
dependencies ([?][rationale-implicit-public]). When a standard library is
explicitly written, then it will be private by default, like any other written
dependency, unless explicitly marked as public ([?][rationale-explicit-private]).

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
```

..is equivalent to the following explicit dependency on `std`:

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { builtin = true, public = true }
```

*See the following sections for rationale/alternatives:*

- [*Why default to public for the implicit standard library dependencies?*][rationale-implicit-public]
- [*Why follow the default privacy of explicit standard library dependencies?*][rationale-explicit-private]

### `rustc_dep_of_std`
[rustc_dep_of_std]: #rustc_dep_of_std

With first-class explicit dependencies on the standard library,
`rustc_dep_of_std` is rendered unnecessary and explicit dependencies on the
standard library can always be present in the `Cargo.toml` of the standard
library's dependencies.

The `core`, `alloc` and `std` dependencies can be patched in the standard
library's workspace to point to the local copy of the crates.

### `dev-dependencies` and `build-dependencies`
[dev-dependencies-and-build-dependencies]: #dev-dependencies-and-build-dependencies

There is no implicit dependency on the standard library in `build-dependencies`
and explicit dependencies on the standard library are not supported
([?][rationale-no-deps-in-build-deps]).

Implicit and explicit dependencies on the standard library are supported for
`dev-dependencies` in the same way as regular `dependencies`. An additional
implicit dependency on the `test` crate is added for `dev-dependencies`.
`test = { builtin = true }` can also be written explicitly.

### Registries
[registries]: #registries-1

Standard library dependencies will be present in the registry index
([?][rationale-cargo-index]). A `builtin_deps` key is added to the
[index's JSON schema][cargo-json-schema] ([?][rationale-cargo-builtindeps]).
`builtin_deps` is similar to the existing `deps` key and contains a list of JSON
objects, each representing a dependency that is "builtin" to the Rust toolchain
and cannot otherwise be found in the registry. 


> [!NOTE]
> 
> It is expected that the keys of these objects will be:
>
> - `name`
>   - String containing name of the `builtin` package. Can shadow the names of
>     other packages in the registry (except those packages in the `deps` key
>     of the current package) ([?][rationale-cargo-index-shadowing])
>
> - `features`:
>   - An array of strings containing enabled features in order to support changing
>     the standard library features on nightly. Optional, empty by default.
> 
> - `optional`, `default_features`, `target`, `kind`:
>   - These keys have the same definition as in the `deps` key.
>
> The keys `req`, `registry` and `package` from `deps` are not required per the
> limitations on builtin dependencies.

The key is optional and its default value will be the implicit builtin
dependencies:

```json
"builtin_deps" : [
    {
        "name": "std",
        "features": [],
        "optional": false,
        "default_features": true,
        "target": null,
        "kind": "normal",
    },
    {
        "name": "alloc",
        ... # as above
    },
    {
        "name": "core",
        ... # as above
    }
]
```

*See the following sections for rationale/alternatives:*

- [*Why add standard library crates to Cargo's index?*][rationale-cargo-index]
- [*Why add a new key to Cargo's registry index JSON schema?*][rationale-cargo-builtindeps]
- [*Why can `builtin_deps` shadow other packages in the registry?*][rationale-cargo-index-shadowing]

## Rebuilding the standard library
[rebuilding-the-standard-library]: #rebuilding-the-standard-library

Cargo configuration will contain a new key `build-std` under the `[profile]`
section ([?][rationale-build-std-in-config]), permitting one of three values -
"off" ([?][rationale-build-std-off]), "target-modifiers" or "match-profile" :

```toml
[profile.dev]
build-std = "target-modifiers" # or `off`/`match-profile`
```

`build-std` defaults to "target-modifiers" for the `dev` profile
([?][rationale-why-not-always-rebuild]) and to "match-profile" for the `release`
profile ([?][rationale-different-defaults]). `test` inherits this from `dev` and
`bench` from `release`.

As the Cargo configuration is local to the current installation of Cargo
(typically in `~/.config/cargo`), the value of `build-std` is not influenced by
the dependencies of the current crate.

In addition, `build-std` can be set in the `[target.<triple>]` and
`[target.<cfg>]` sections. If set, this takes precedence over the configuration
in `[profile]`.

Cargo will use the pre-built standard library automatically depending on the
value of the `build-std` key ([?][rationale-why-automatic]):

- If `build-std = "off"`, then the pre-built standard library artifact is always
  used. If it is not present or is incompatible with the rest of the crate graph
  (due to target modifiers), rustc will emit an error.

  Profile changed/customised? | Target modifiers changed? | Standard library re-built?
  --------------------------- | ------------------------- | ------------------------------
  No                          | No                        | No
  Yes                         | No                        | No
  Yes                         | Yes                       | Error!
- If `build-std = "target-modifiers"`, then the pre-built standard library will
  be used as long as it was compiled with target modifiers compatible with the
  current profile.

  Profile changed/customised? | Target modifiers changed? | Standard library re-built?
  --------------------------- | ------------------------- | ------------------------------
  No                          | No                        | No
  Yes                         | No                        | No
  Yes                         | Yes                       | Yes
- If `build-std = "match-profile"`, then the pre-built standard library will be
  used only if it has an identical configuration to the current profile.
  
  Profile changed/customised? | Target modifiers changed? | Standard library re-built?
  --------------------------- | ------------------------- | ------------------------------
  No                          | No                        | No
  Yes                         | No                        | Yes
  Yes                         | Yes                       | Yes

When the pre-built standard library is not used or available, Cargo will build
and use the standard library from source with the requested profile.

> [!NOTE]
>
> Inspired by the concept of [opaque dependencies][Opaque dependencies], the
> dependencies of the standard library crates are entirely opaque to the user,
> who cannot control compilation any of the dependencies of the `core`, `alloc`
> or `std` standard library crates individually.
>
> The lockfile included in the standard library source will be used when
> resolving the standard library's dependencies ([?][rationale-lockfile]).
>
> The standard library will always be a non-incremental build
> ([?][rationale-incremental]), with no `depinfo` produced, and only a `rlib`
> produced (no `dylib`) ([?][rationale-no-dylib]). It will be built into the
> `target` directory of the crate or workspace like any other dependency.

> [!NOTE]
>
> A key detail in build-std is the contract between rustc and Cargo that must be
> upheld when Cargo is providing standard library dependencies:
>
> rustc only requires that `--extern noprelude:std=path/to/std.lib`,
> `--extern noprelude:alloc=path/to/alloc.lib` and
> `--extern noprelude:core=path/to/core.lib` be passed, assuming all three
> crates are required. No other `--extern` flags are required.
>
> `--extern noprelude:alloc=path/to/alloc.lib` and
> `--extern noprelude:core=path/to/core.lib` are only required in addition to
> `--extern noprelude:std=path/to/std.lib` so that writing `extern crate alloc`
> or `extern crate core` works as it does with the pre-built standard library
> from the sysroot.
>
> rustc will find any dependencies of these crates first from the paths provided
> with `-L dependency=` and then from the sysroot. rustc will attempt to load
> some crates like `compiler_builtins` and `panic_unwind` itself.
>
> rustc will need patched to be able to load `panic_unwind` from
> `-L dependency=` paths.
>
> Cargo will load the sysroot crate in the standard library and perform the
> resolve on that workspace with the packages that the user has a explicit
> dependency on and those which require `--extern` to be passed explicitly.

The host pre-built standard library will always be used for procedural macros
and build scripts ([?][rationale-sysroot-for-host-deps]). Artifact dependencies
use the same standard library as the rest of the crate (pre-built or
newly-built, as appropriate).

*See the following sections for rationale/alternatives:*

- [*Why put `build-std` in the Cargo config?*][rationale-build-std-in-config]
- [*Why not always rebuild when the profile changes?*][rationale-why-not-always-rebuild]
- [*Why have different build-std defaults depending on the profile?*][rationale-different-defaults]
- [*Why rebuild the standard library automatically?*][rationale-why-automatic]
- [*Why use the lockfile of the `rust-src` component?*][rationale-lockfile]
- [*Why not build the standard library in incremental?*][rationale-incremental]
- [*Why not produce a `dylib` for the standard library?*][rationale-no-dylib]
- [*Why use the pre-built standard library for procedural macros and build-scripts?*][rationale-sysroot-for-host-deps]

*See the following sections for relevant unresolved questions:*

- [*Where should the `build-std` configuration in `.cargo/config` be and what should it be called?*][unresolved-config-location-name]
- [*What should the values of the `build-std` config be named?*][unresolved-config-values]

### Profiles
[profiles]: #profiles

Cargo will assume that the pre-built standard library matches the standard
library's release profile ([?][rationale-assume-release-profile]). If the user
changes the default release profile or builds with a different profile then this
could trigger a rebuild of the standard library
([?][rationale-ship-debug-std]), depending on the value of the `build-std`
config as above.

User's Cargo profile | Target modifiers changed? | Standard library profile
-------------------- | ------------------------- | ------------------------
`release`            | No                        | `release` (pre-built)
`dev`                | No                        | `release` (pre-built)
`release`            | Yes                       | `release` (newly built)
`dev`                | Yes                       | `dev` (newly built)

> [!NOTE]
>
> If `build-std` is set to `target-modifiers`, Cargo must decide if `build-std`
> should be enabled. rustc will add a `--print target-modifiers` flag which will
> print all of the flags treated as target modifiers, like `-Zretpoline`, with
> one flag per line and its default value.
>
> If changing a profile configuration would result in one of these flags being
> emitted by Cargo then it assumes a target modifier has changed from the
> default release profile and would no longer match the pre-built standard
> library.

Unlike other dependencies, the profiles defined in the standard library's
workspace will apply to its build even when used as a dependency
([?][rationale-respect-std-profile]) and the user's profile configuration will
override it only where the user's profile differs from its default.

When rebuilt, standard library crates will be built using the configuration of
the current profile as defined in the standard library's workspace. For example,
if using the `release` profile and the standard library needs to be rebuilt,
then the release profile of the standard library workspace will be used.

If the user customises their profile from its defaults, then the modified
options will be merged with the standard library's profile
([?][rationale-why-merge]). For example, if the user sets
`profile.release.opt-level` that will override the standard library's release
`opt-level` and if the user sets `profile.release.rustflags` that will be
appended to the standard library's release `rustflags`. Merging behaviour for
profile fields will be determined by the type of the field (e.g. lists are
appended and strings/integers overridden).

Profile overrides in the standard library's workspace continue to apply to its
dependencies ([?][rationale-respect-profile-overrides]). User profile overrides
for specific crates can only apply to the `std`, `alloc` and `core` crates
([?][rationale-why-not-override-std-deps]).

Changes to the rustc options passed to the `std`, `alloc` and `core` crates also
apply to their dependencies.

> [!NOTE]
>
> As much as is possible, the configuration of the pre-built standard library
> will be moved into the profile configuration of the standard library workspace
> and crates. This will enable Cargo's heuristics about when to rebuild the
> standard library to match the pre-built standard library as closely as
> possible.

*See the following sections for rationale/alternatives:*

- [*Why default to assuming the pre-built standard library is the release profile?*][rationale-assume-release-profile]
- [*Why not ship a debug profile `rust-std`?*][rationale-ship-debug-std]
- [*Why respect the profile of the standard library workspace?*][rationale-respect-std-profile]
- [*Why merge the user's profile and the standard library workspace's profile?*][rationale-why-merge]
- [*Why respect profile overrides of the standard library's workspace?*][rationale-respect-profile-overrides]
- [*Why not allow profile overrides to override the standard library's dependencies?*][rationale-why-not-override-std-deps]

### Preventing implicit sysroot dependencies
[preventing-implicit-sysroot-dependencies]: #preventing-implicit-sysroot-dependencies

Cargo will pass a new flag to rustc which will prevent rustc from loading
top-level dependencies from the sysroot ([?][rationale-root-sysroot-deps]).

> [!NOTE]
>
> rustc could add a `--no-implicit-sysroot-deps` flag with this behaviour. For
> example, writing `extern crate foo` in a crate will not load `foo.rlib` from
> the sysroot if it is present, but if an `--extern noprelude:bar.rlib` is
> provided which depends on a crate `foo`, rustc will look in `-L` paths and the
> sysroot for it.

All Cargo dependencies are provided to the compiler using the
`--extern noprelude:` flag ([?][rationale-noprelude-with-extern]), including
explicit and implicit standard library dependencies.

*See the following sections for rationale/alternatives:*

- [*Why prevent rustc from loading root dependencies from the sysroot?*][rationale-root-sysroot-deps]
- [*Why use `noprelude` with `--extern`?*][rationale-noprelude-with-extern]

### Vendored `rust-src`
[vendored-rust-src]: #vendored-rust-src

When it is necessary to build the standard library, Cargo will look for sources
in a fixed location in the sysroot ([?][rationale-custom-src-path]):
`lib/rustlib/src`. rustup's `rust-src` component downloads standard library
sources to this location. If the sources are not found, Cargo will emit an error
and recommend the user download `rust-src` if using rustup.

`rust-src` will contain the sources for the standard library crates as well as
its vendored dependencies ([?][rationale-vendoring]). Sources of standard
library dependencies will not be fetched from crates.io.

> [!NOTE]
>
> Cargo will not perform any checks to ensure that the sources in `rust-src`
> have been modified ([?][rationale-src-modifications]). It will be documented
> that modifying these sources is not supported.

*See the following sections for rationale/alternatives:*

- [*Why not allow the source path for the standard library be customised?*][rationale-custom-src-path]
- [*Why vendor standard library dependencies?*][rationale-vendoring]
- [*Why not check if `rust-src` has been modified?*][rationale-src-modifications]

### Building the standard library on a stable toolchain
[building-the-standard-library-on-a-stable-toolchain]: #building-the-standard-library-on-a-stable-toolchain

rustc will automatically assume `RUSTC_BOOTSTRAP` when the source path of the
crate being compiled is within the same sysroot as the rustc binary being
invoked ([?][rationale-implied-bootstrap]). Cargo will not need to use
`RUSTC_BOOTSTRAP` when compiling the standard library with a stable toolchain.

*See the following sections for rationale/alternatives:*

- [*Why allow building from the sysroot with implied `RUSTC_BOOTSTRAP`?*][rationale-implied-bootstrap]

### Panic strategies
[panic-strategies]: #panic-strategies-1

Panic strategies are unlike other profile settings insofar as they influence
which crates and flags are passed to the standard library. For example, if
`panic = "unwind"` were set in the Cargo profile then the `panic_unwind` feature
would need to be provided to `std` and `-Cpanic=unwind` passed to suggest that
the compiler use that panic runtime.

If the current crate has no dependency on `std` (i.e. have added an `alloc` or
`core` dependency explicitly to opt-out of the implicit `std` dependency), then
Cargo will not build either of the `panic_unwind` or `panic_abort` crates or
pass `-Cpanic` to rustc. In this circumstance, if `panic` is set in the Cargo
profile, then this value will be ignored and Cargo will emit a warning informing
the user of this.

If the crate does depend on `std`, then Cargo's behaviour depends on whether or
not `panic` is set in the profile:

- If `panic` is not set in the profile then unwinding may still be the default
  for the target and Cargo will need to enable the `panic_unwind` feature to the
  standard library just in case it is used.
- If `panic` is set to "unwind" then the `panic_unwind` feature will be enabled
  and `-Cpanic=unwind` will be passed.
- If `panic` is set to "abort" then `-Cpanic=abort` will be passed.
  - `panic_abort` is a non-optional dependency of `std` so it will always be
    built.

Tests, benchmarks, build scripts and proc macros continue to ignore the "panic"
setting and `panic = "unwind"` is always used - which means the standard library
needs to be recompiled again if the user is using "abort". Once
`panic-abort-tests` is stabilised, the standard library can be built with the
profile's panic strategy even for tests and benchmarks.

Cargo will not inspect the `RUSTFLAGS` environment variable for compilation
flags that would require additional crates to be built for compilation to
succeed.

*See the following sections for future possibilities:*

- [*Avoid building `panic_unwind` unnecessarily*][future-panic_unwind]

### Special object files
[special-object-files]: #special-object-files

A handful of targets require linking against special object files, such as
`windows-gnu`, `linux-musl` and `wasi` targets. For example, `linux-musl`
targets require `crt1.o`, `crti.o`, `crtn.o`, etc.

Since [rust#76185]/[compiler-team#343], the compiler has a stable
`-Clink-self-contained` flag which will look for special object files in
expected locations, typically populated by the `rust-std` components. Its
behaviour can be forced by `-Clink-self-contained=true`, but is force-enabled
for some targets and inferred for others.

Rust can start to ship `rust-self-contained-$target` components for any targets
which need it (including tier three targets). These components will contain the
special object files normally included in `rust-std`, and will be distributed
for all tiers of targets. While generally these objects are specific to the
architecture and C runtime (CRT) (and so `rust-self-contained-$arch-$crt` could
be sufficient and result in fewer overall components), it's technically possible
that Rust could support two targets with the same architecture and same CRT but
different versions of the CRT, so having target-specific components is most
future-proof. These would replace the `self-contained` directory in existing
`rust-std` components.

As long as these components have been downloaded, as well as any other support
components, such as `rust-mingw`, rustc's `-Clink-self-contained` will be able
to link against the object files and build-std should never fail on account of
missing special object files.

*See the following sections for future possibilities:*

- [*Enable local recompilation of special object files/sanitizer runtimes*][future-recompile-special]

### `compiler-builtins-mem`
[compiler-builtins-mem]: #compiler-builtins-mem

The `mem` feature of `compiler_builtins` (and the subsequent
`compiler-builtins-mem` feature of `core`, `alloc`, `std` which forward to
`compiler_builtins/mem`) is required by `no_std` crates because `libc` does not
provide these symbols without `std`.

It is necessary that the `compiler-builtins-mem` feature of `alloc` and/or
`core` be enabled when `std` is not in the crate graph
([?][rationale-no-weak-linkage]).

*See the following sections for rationale/alternatives:*

- [*Why not use weak linkage for `compiler-builtins/mem` symbols?*][rationale-no-weak-linkage]

### `libunwind`
[libunwind]: #libunwind

`libunwind`'s sources are included in the `rust-src` component so that they can
be used as part of the standard library build on targets which require it.

### Potential migration breakage
[potential-migration-breakage]: #potential-migration-breakage

When building an existing `no_std` project for a tier two or three target with
build-std, there could be an implicit dependency on `std` from a dependency
`no_std` crate that has not yet made its dependency on only the `core` crate
explicit, for example. In this circumstance, this would fail to build as the
target will have had `standard_library_support.std = false` in its target
specification and Cargo will refuse to build `std` (see
[*Target standard library support*][target-standard-library-support])
([?][rationale-breakage]).

*See the following sections for rationale/alternatives:*

- [*Why permit breakage of nightly build-std users using tier three targets?*][rationale-breakage]

### Caching
[caching]: #caching

Standard library artifacts built by build-std will not be shared between crates
or workspaces, as they only exist in the `target` directory of a specific crate
or workspace ([?][rationale-caching]).

*See the following sections for rationale/alternatives:*

- [*Why not globally cache builds of the standard library?*][rationale-caching]

### Sanitizers
[sanitizers]: #sanitizers

rustc's sanitizer support is currently unstable as it is not possible for users
to re-build the standard library with sanitizer support.

It is out-of-scope for this RFC to propose stabilising sanitizers (see
[rust#123617]) or to expose sanitizer configuration in Cargo, but it is
instructive to examine how build-std would enable sanitizer support to ensure
that the proposed design is compatible.

rustc's flag to enable sanitizers will be a target modifier, as the
instrumentation must be present for all of the crates to avoid false negatives.

rustc's sanitizer support attempts to locate sanitizer runtimes in the sysroot
(`$sysroot/lib/rustlib/$target/lib/`) to link against. Rust already ships
sanitizer runtimes for targets that support sanitizers and with the use of
`build-std` as proposed in this RFC all Rust crates in a crate graph should have
coverage with the requested sanitizers.

Combining these shipped sanitizer runtimes with other target modifiers is
outside the scope of this RFC.

## Cargo subcommands
[cargo-subcommands]: #cargo-subcommands

As opaque dependencies, any Cargo command which accepts a package spec with `-p`
will only additionally recognise `core`, `alloc` and `std` and none of their
dependencies. Many of Cargo's subcommands will need modification to support
build-std:

[`cargo add`][cargo-add] will add `core`, `alloc` or `std` explicitly to the
manifest if invoked with those crate names:

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { builtin = true } # <-- this would be added
```

[`cargo clean`][cargo-clean] will additionally delete any builds of the standard
library performed by build-std.

[`cargo fetch`][cargo-fetch] will not fetch the standard library dependencies as
they are already vendored in the `rust-src` component.

[`cargo info`][cargo-info] will learn how to print information for the built-in
`std`, `alloc` and `core` dependencies:

```shell-session
$ cargo info std
std
rust standard library
license: Apache 2.0 + MIT
rust-version: 1.86.0
documentation: https://doc.rust-lang.org/1.86.0/std/index.html
```

```shell-session
$ cargo info alloc
alloc
rust standard library
license: Apache 2.0 + MIT
rust-version: 1.86.0
documentation: https://doc.rust-lang.org/1.86.0/alloc/index.html
```

```shell-session
$ cargo info core
core
rust standard library
license: Apache 2.0 + MIT
rust-version: 1.86.0
documentation: https://doc.rust-lang.org/1.86.0/core/index.html
```

[`cargo metadata`][cargo-metadata] will emit `std`, `alloc` and `core`
dependencies to the metadata emitted by `cargo metadata` (when those crates are
dependencies). None of the standard library's dependencies will be included.
`source` would be set to `builtin` and the remaining fields would be set like
any other dependency.

> [!NOTE]
>
> `cargo metadata` output could look as follows:
>
> ```json
> {
>   "packages": [
>     {
>       /* ... */
>       "dependencies": [
>         {
>           "name": "std",
>           "source": "builtin",
>           "req": "*",
>           "kind": null,
>           "rename": null,
>           "optional": false,
>           "uses_default_features": true,
>           "features": ["compiler-builtins-mem"],
>           "target": null,
>           "public": truee
>         }
>       ],
>       /* ... */
>     }
>   ]
> }               
> ```

[`cargo miri`][cargo-miri] is not built into Cargo, it is shipped by miri, but
is mentioned in Cargo's documentation. `cargo miri` is unchanged by this RFC,
but build-std is one step towards `cargo miri` requiring less special support.

> [!NOTE]
>
> `cargo miri` could be re-implemented using build-std to enable a `miri`
> profile and always rebuild. The `miri` profile would be configured in the
> standard library's workspace, setting the flags/options necessary for `miri`.

[`cargo pkgid`][cargo-pkgid] when passed `-p core` would print `builtin#core` as
the source, likewise with `alloc` and `std`.

[`cargo report`][cargo-report] will not include reports from the standard
library crates or their dependencies.

[`cargo remove`][cargo-remove] will remove `core`, `alloc` or `std` explicitly
from the manifest if invoked with those crate names:

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
std = { builtin = true } # <-- this would be removed
```

[`cargo tree`][cargo-tree] will show `std`, `alloc` and `core` at appropriate
places in the tree of dependencies. `alloc` will always be shown as a dependency
of `std`, and `core` a dependency of `alloc`. As opaque dependencies, none of
the other dependencies of `std`, `alloc` or `core` will be shown. Neither `std`,
`alloc` or `core` will have a version number.

> [!NOTE]
>
> `cargo tree` output could look as follows:
>
> ```shell-session
> $ cargo tree
> myproject v0.1.0 (/myproject)
> ├── rand v0.7.3
> │   ├── getrandom v0.1.14
> │   │   ├── cfg-if v0.1.10
> │   │   │   └── core v0.0.0
> │   │   ├── libc v0.2.68
> │   │   │   └── core v0.0.0
> │   │   └── core v0.0.0
> │   ├── libc v0.2.68 (*)
> │   │   └── core v0.0.0
> │   ├── rand_chacha v0.2.2
> │   │   ├── ppv-lite86 v0.2.6
> │   │   │   └── core v0.0.0
> │   │   ├── rand_core v0.5.1
> │   │   │   ├── getrandom v0.1.14 (*)
> │   │   │   └── core v0.0.0
> │   │   └── std v0.0.0
> │   │       └── alloc v0.0.0
> │   │           └── core v0.0.0
> │   ├── rand_core v0.5.1 (*)
> │   └── std v0.0.0 (*)
> └── std v0.0.0 (*)
> ```

[`cargo update`][cargo-update] will not update the dependencies of `std`,
`alloc` and `core`, as these are vendored as part of the distribution of
`rust-src` and resolved separately from the user's dependencies. Neither will
`std`, `alloc` or `core` be updated, as these are unversioned and always match
the current toolchain version.

[`cargo vendor`][cargo-vendor] will not vendor standard library dependencies.
Vendoring these and using them later would effectively pin the crate to the
version of the language and toolchain used when vendoring was performed (as the
vendored standard library source would only work with that toolchain version).
Standard library crates are already vendored in the `rust-src` component, so do
not require network access once downloaded.

The following commands will now build the standard library if required as part
of the compilation of the project, just like any other dependency:

- [`cargo bench`][cargo-bench]
- [`cargo build`][cargo-build]
- [`cargo check`][cargo-check]
- [`cargo clippy`][cargo-clippy]
- [`cargo doc`][cargo-doc]
- [`cargo fix`][cargo-fix]
- [`cargo run`][cargo-run]
- [`cargo rustc`][cargo-rustc]
- [`cargo rustdoc`][cargo-rustdoc]
- [`cargo test`][cargo-test]

build-std has no implications for the following Cargo subcommands:

- [`cargo fmt`][cargo-fmt]
- [`cargo generate-lockfile`][cargo-generate-lockfile]
- [`cargo help`][cargo-help]
- [`cargo init`][cargo-init]
- [`cargo install`][cargo-install]
- [`cargo locate-project`][cargo-locate-project]
- [`cargo login`][cargo-login]
- [`cargo logout`][cargo-logout]
- [`cargo new`][cargo-new]
- [`cargo owner`][cargo-owner]
- [`cargo package`][cargo-package]
- [`cargo publish`][cargo-publish]
- [`cargo search`][cargo-search]
- [`cargo uninstall`][cargo-uninstall]
- [`cargo version`][cargo-version]
- [`cargo yank`][cargo-yank]

## Constraints on the standard library, compiler and bootstrap
[constraints-on-the-standard-library]: #constraints-on-the-standard-library-compiler-and-bootstrap

A stable mechanism for building the standard library imposes some constraints on
the rest of the toolchain that would need to be upheld:

- No further customisation of the pre-built standard library through any means
  other than the profile in `Cargo.toml`
- No new C dependencies on the standard library

> [!NOTE]
>
> Cargo could be made a [JOSH] subtree of the [rust-lang/rust] so that all
> relevant parts of the toolchain can be updated in tandem when this is
> necessary.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This section aims to justify all of the decisions made in the proposed design
from [*Detailed explanation*][detailed-explanation] and discuss why alternatives
were not chosen.

## Proposal-wide
[rationale-proposal-wide]: #proposal-wide

These rationales and alternatives apply to the proposal as-a-whole, rather than
any specific section:

### Why not do nothing?
[rationale-why-not-do-nothing]: #why-not-do-nothing

Support for rebuilding the standard library is a long-standing feature request
from subsets of the Rust community and blocks the work of some project teams
(e.g. sanitisers and branch protection in the compiler team, amongst others).
Inaction forces these users to remain on nightly and depend on the unstable
`-Zbuild-std` flag indefinitely. RFCs and discussion dating back to the first
stable release of the language demonstrate the longevitity of build-std as a
need.

### Shouldn't build-std be part of rustup?
[rationale-in-rustup]: #shouldnt-build-std-be-part-of-rustup

build-std is effectively creating a new sysroot with a customised standard
library. rustup as Rust's toolchain manager has lots of existing machinery
to create and maintain sysroots. rustup knows how to download `rust-src`, it
knows how to create a new toolchain from an existing sysroot (as in
`rustup toolchain link`), it would only need to learn how to invoke Cargo on the
`rust-src` sources. rustup would be invoking tools from the next layer of
abstraction (Cargo) in the same way that Cargo invokes tools from the layer of
abstraction after it (rustc).

A brief prototype of this idea was created and a
[short design document was drafted][why-not-rustup] before concluding that it
would not be possible. With artifact dependencies, it may be desirable to build
with a different standard library and if rustup was creating different
toolchains per-customised standard library then Cargo would need to have
knowledge of these to switch between them, which isn't possible (and something
of a layering violation). It is also unclear how Cargo would find and use the
uncustomised host sysroot for build scripts and procedural macros.

### Why not replace `#![no_std]` as the source-of-truth for whether a crate depends on `std`?
[rationale-replace-no_std]: #why-not-replace-no_std-as-the-source-of-truth-for-whether-a-crate-depends-on-std

Crates can currently use the crate attribute `#![no_std]` to indicate a lack of
dependency on `std`. With `Cargo.toml` being used to express a dependency on the
standard library (or lack thereof), it is unintuitive for there to be two
sources-of-truth for this information.

`#![no_std]` serves two purposes - it stops the compiler from loading `std` from
the sysroot and adding `extern crate std`, and it prevents the user from
depending on anything from `std` accidentally.

`#![no_std]` could hypothetically be replaced by a lint to prevent use of the
standard library and a change to the compiler so that it loads the `std`
speculatively unless it is used.

However, while rustc does have some support for speculatively loading crates, it
is not possible to do so and not declare them as a dependency in cross-crate
metadata.

## Target standard library support
[rationale-target-standard-library-support]: #target-standard-library-support-1

These rationale/alternatives apply to the content in the
[*Target standard library support*][target-standard-library-support] section.

### Should target specifications own knowledge of which standard library crates are supported?
[rationale-target-spec-purpose]: #should-target-specifications-own-knowledge-of-which-standard-library-crates-are-supported

It is much simpler to record this information in a target's specification than
to try and match on the target's cfg values in a `build.rs`, set a cfg and then
emit an error in the source code.

Target specifications have typically been considered part of the compiler and
there has been hesistation to have target specs be the source of truth for
information like standard library support, as this is the domain of the library
team and ought to be owned by the standard library (such as in the standard
library's `build.rs`). However, with appropriate processes and sync points,
there is no reason why the target specification could not be primarily
maintained by t-compiler but in close coordination with library and other
relevant teams.

↩ [*Target standard library support*][target-standard-library-support]

### Why record support for `core`, `alloc` and `std` separately?
[rationale-target-spec-core-alloc-std]: #why-record-support-for-core-alloc-and-std-separately

It is intuitive that some targets may not support the standard library and so
needing to keep track of whether `std` is supported is necessary. However, it is
not obvious why keeping track of whether `alloc` and `core` are supported
individually is necessary:

Most targets will support `core`. `core` would only be set to `false` for very
experimental targets which do not support build-std at all. `alloc` would be set
to `false` for those targets that do not support allocation.

↩ [*Target standard library support*][target-standard-library-support]

### Why replace `restricted_std` with explicit standard library support for a target?
[rationale-replace-restricted-std]: #why-replace-restricted_std-with-explicit-standard-library-support-for-a-target

`restricted_std` was originally added as part of a mechanism to enable the
standard library to build on all targets (just with stubbed out functionality),
however stability is not an ideal match for this use case. When `restricted_std`
applies, users must add `#![feature(restricted_std)]` to opt-in to using the
standard library anyway (conditionally, only for affected targets), and have no
mechanism for opting-in on behalf of their dependencies (including first-party
crates like `libtest`).

↩ [*Target standard library support*][target-standard-library-support]

### Why disallow custom targets?
[rationale-disallow-custom-targets]: #why-disallow-custom-targets

While custom targets can be used on stable today, in practice, they are only
used on nightly as `-Zbuild-std` would need to be used to build at least `core`.
As such, if build-std were to be stabilised, custom targets would become much
more usable on stable toolchains.

In order to avoid users relying on the [unstable target-spec-json][rust#71009] format on a
stable toolchain, using custom targets with build-std on a stable toolchain is
disallowed by Cargo until another RFC can consider all the implications of this
thoroughly. The idea of rustc disallowing custom targets on stable is covered
in [rust#71009].

↩ [*Custom targets*][custom-targets]

## Standard library dependencies
[rationale-standard-library-dependencies]: #standard-library-dependencies-1

These rationale/alternatives apply to the content in the
[*Standard library dependencies*][standard-library-dependencies] section.

### Why explicitly declare dependencies on the standard library in `Cargo.toml`?
[rationale-why-explicit-deps]: #why-explicitly-declare-dependencies-on-the-standard-library-in-cargotoml

If there are no explicit dependencies on standard library crates, Cargo would
need to be able to determine which standard library crates to build when this is
required:

- Cargo could unconditionally build `std`, `alloc` and `core`. Not only would
  this be unnecessary and wasteful for `no_std` crates in the embedded
  ecosystem, but sometimes a target may not support building `std` at all and
  this would cause the build to fail.
- rustc could support a `--print` value that would print whether the crate
  declares itself as `#![no_std]` crate, and based on this, Cargo could build
  `std` or only `core`. This would require asking rustc to parse crates'
  sources while resolving dependencies, slowing build times. Alternatively,
  Cargo can already read Rust source to detect frontmatter (for `cargo script`)
  so it could additionally look for `#![no_std]` itself. Regardless of how it
  determines a crate is no-std, Cargo would also need to know whether to build
  `alloc` too, which checking for `#![no_std]` does not help with. Cargo could
  go further and ask rustc whether a crate (or its dependencies) used `alloc`,
  but this seems needlessly complicated.

Furthermore, supporting explicit dependencies on standard library crates enables
use of other Cargo features that apply to dependencies in a natural and
intuitive way. If there were not explicit standard library dependencies and
enabling features on the `std` crate was desirable, then a mechanism other than
the standard syntax for this would be necessary, such as a flag (e.g.
`-Zbuild-std-features`) or option in Cargo's configuration. This also applies to
optional dependencies, public/private features, etc.

See
[*Why rebuild the standard library automatically?*][rationale-why-automatic] for
a larger look at alternative user experiences to the build-std.

↩ [*Standard library dependencies*][standard-library-dependencies]

### Why disallow builtin dependencies to be combined with other sources?
[rationale-builtin-other-sources]: #why-disallow-builtin-dependencies-to-be-combined-with-other-sources

Combining `path`/`git` sources with `builtin` dependencies would enable crates
with `path`/`git` standard library dependencies to be pushed to crates.io -
assuming it were to work like combining `path`/`git` dependencies with crates.io
sources using `version`.

This is not desirable as it is unclear that supporting `path`/`git` sources
which shadow standard library crates was a deliberate choice and so enabling
that pattern to be used more widely when not necessary is needlessly permissive.

When combined with a `git`/`path` source, the `version` key will also check the
requirement against the version of the local package. This behaviour of the key
is a poor fit for `builtin` dependencies for a number of reasons:

- The `std`, `alloc` and `core` crates all currently have a version of `0.0.0`
- Choosing different version requirements for different `builtin` crates has no
  purpose

Equivalent behaviour is handled by the `rust-version` key (which represents the
minimum supported Rust version) and allows resolvers with support for the key to
avoid choosing packages that do not support the current toolchain version.

↩ [*Standard library dependencies*][standard-library-dependencies]

### Why disallow builtin dependencies on other crates?
[rationale-no-builtin-other-crates]: #why-disallow-builtin-dependencies-on-other-crates

`builtin` dependencies could be accepted on two other crates - dependencies of
the standard library, like `compiler_builtins`, or other crates in the sysroot
added manually by users:

- The standard library's dependencies are not part of the stable interface of
  the standard library and it is not desirable that users can observe their
  existence or depend on them directly.
- Other crates in the sysroot added by users are not something that can
  reasonably be supported by build-std and should be added as regular
  dependencies.

↩ [*Standard library dependencies*][standard-library-dependencies]

### Why imply direct builtin dependencies?
[rationale-builtin-implied-direct]: #why-imply-direct-builtin-dependencies

Cargo passes direct dependencies of the current crate with the `--extern` flag
and passes the `-L dependency=...` flag so rustc can search for transitive
dependencies itself. Looking for direct dependencies in a `-L crate=...`
directory would create the possibility of rustc finding stale artifacts from
previous builds. As a consequence, Cargo must be aware of the names of any
direct dependencies of a crate and cannot rely on the fact that they are part of
the dependency graph.

A possible alternative is for Cargo to require all `builtin` dependencies to be
explicit but validate the hierarchy to ensure users always include a `core`
dependency. This is needlessly verbose.

↩ [*Standard library dependencies*][standard-library-dependencies]

### Why not migrate to always requiring explicit standard library dependencies?
[rationale-no-migration]: #why-not-migrate-to-always-requiring-explicit-standard-library-dependencies

Explicit standard library dependencies with `builtin = true` will necessarily
only be understood by newer versions of Cargo.

If all packages were required to add explicit dependencies (perhaps over an
edition or through some other mechanism), then every crate would require the
newest version of Cargo to be understood, effectively raising the MSRV of every
Rust crate.

If only `no_std` crates (or crates with a `std` feature) add explicit
dependencies on `core` or `alloc` then a much smaller percentage of the crates
ecosystem will require the newest Cargo versions for their new explicit standard
library dependencies to be understood.

Alternative syntaxes, such as requiring `version = "*"` for explicit standard
library dependencies, could be worthwhile to maintain a greater level of
compatibility with older toolchain versions. Any currently accepted syntax would
necessarily be interpreted differently by the build-std-supporting versions of
Cargo, so this approach has its own complications. For example, while
`version = "*"` would be understood by older versions of Cargo, it would attempt
to find the standard library crates on crates.io and fail unless empty crates
were published named `core`, `alloc` and `std`. This is not a build-std specific
issue and is true of any RFC adding to what can be written in `Cargo.toml`.

↩ [*Standard library dependencies*][standard-library-dependencies]

### Why disallow renaming standard library dependencies?
[rationale-package-key]: #why-disallow-renaming-standard-library-dependencies

Cargo allows [renaming dependencies][cargo-docs-renaming] with the `package`
key, which allows user code to refer to dependencies by names which do not
match their `package` name in their respective `Cargo.toml` files.

However, rustc expects the standard library crates to be present with their
existing names - for example, `core` is always added to the [extern prelude][rust-extern-prelude].
This feature would not work without a way to tell rustc the new names of
`builtin` crates.

↩ [*Standard library dependencies*][standard-library-dependencies]

### Why add standard library dependencies to `Cargo.lock`?
[rationale-cargo-lock]: #why-add-standard-library-dependencies-to-cargolock

`Cargo.lock` is a direct serialisation of a resolve and that must be a two-way
non-lossy process in order to make the `Cargo.lock` useful without doing further
resolution to fill in missing `builtin` packages.

↩ [*Standard library dependencies*][standard-library-dependencies]

### Why permit patching of the standard library dependencies on nightly?
[rationale-patching]: #why-permit-patching-of-the-standard-library-dependencies-on-nightly

Being able to patch `builtin = true` dependencies and replace their source with
a `path` dependency is required to be able to replace `rustc_dep_of_std`. As
crates which use these sources cannot be published to crates.io, this would not
enable a usable general-purpose mechanism for crates to modify the standard
library sources. This capability is restricted to nightly as that is all that is
required for it to be used in replacing `rustc_dep_of_std`.

↩ [*Patches*][patches]

### Why limit enabling standard library features to nightly?
[rationale-features]: #why-limit-enabling-standard-library-features-to-nightly

If it were possible to enable features of the standard library crates on stable
then all of the standard library's current features would immediately be held to
the same stability guarantees as the rest of the standard library, which is not
desirable. See
[*Allow enabling/disabling features with build-std*][future-features]

↩ [*Features*][features]

### Why default to public for the implicit standard library dependencies?
[rationale-implicit-public]: #why-default-to-public-for-the-implicit-standard-library-dependencies

There are crates building on stable which re-export from the standard library.
If the implicit standard library dependency were not public then these crates
would start to trigger the `exported_private_dependencies` lint when upgrading
to a version of Cargo with an implicit standard library dependency.

↩ [*Public and private dependencies*][public-and-private-dependencies]

### Why follow the default privacy of explicit standard library dependencies?
[rationale-explicit-private]: #why-follow-the-default-privacy-of-explicit-standard-library-dependencies

This may be unintuitive when a user first writes an explicit standard library
dependency, triggering the `exported_private_dependency` lint, but this would be
caught immediately by the user. However, it is also unintuitive that the default
for privacy of a explicitly written dependency would depend on which crate the
dependency was (i.e. the standard library has a different default than
everything else).

↩ [*Public and private dependencies*][public-and-private-dependencies]

### Why not support implicit or explicit standard library dependencies in `build-dependencies`?
[rationale-no-deps-in-build-deps]: #why-not-support-implicit-or-explicit-standard-library-dependencies-in-build-dependencies

`build-dependencies` only apply to build scripts which are run on the host
toolchain. There is little advantage to using a custom standard library with
build scripts as they are not part of the final output artifact and anywhere
they can run already has a toolchain with host tools and a pre-built standard
library.

See also
[*Why use the pre-built standard library for procedural macros and build-scripts?*][rationale-sysroot-for-host-deps].

↩ [*`dev-dependencies` and `build-dependencies`*][dev-dependencies-and-build-dependencies]

### Why add standard library crates to Cargo's index?
[rationale-cargo-index]: #why-add-standard-library-crates-to-cargos-index

When Cargo builds the dependency graph, it is driven by the index (not
`Cargo.toml`), so builtin dependencies need to be included in the index.

↩ [*Registries*][registries]

### Why add a new key to Cargo's registry index JSON schema?
[rationale-cargo-builtindeps]: #why-add-a-new-key-to-cargos-registry-index-json-schema

Cargo's [registry index schema][cargo-json-schema] is versioned and making a
behaviour-of-Cargo-modifying change to the existing `deps` keys would be a
breaking change. Each packages is published under one particular version of the
schema, meaning that older versions of Cargo cannot use newer versions of
packages which are defined using a schema it does not have knowledge of.

Cargo ignores packages published under an unsupported schema version, so older
versions of Cargo cannot use newer versions of packages relying on these
features. New schema version is disruptive to users on older toolchains and
should be avoided where possible.

Some new fields, including `rust-version`, were added to all versions of the
schema. Cargo ignores fields it does not have knowledge of, so older versions of
Cargo will simply not use `rust-version` and its presence does not change their
behaviour.

Existing versions of Cargo already function correctly without knowledge of
crate's standard library dependencies. A new top-level key will be ignored by
older versions of Cargo, while newer versions will understand it. This is a
different approach to that taken when artifact dependencies were added to the
schema, as those do not have a suitable representation in older versions of
Cargo.

The obvious alternative to a `builtin_deps` key is to modify `deps` entries with
a new `builtin: bool` field and to increment the version of the schema. However,
these entries would not be understood by older versions of Cargo which would
look in the registry to find these packages and fail to do so.

That approach could be made to work if dummy packages for `core`/`alloc`/`std`
were added to registries. Older versions of Cargo would pass these to rustc
via `--extern` and shadow the real standard library dependencies in the sysroot,
so these packages would need to contain `extern crate std; pub use std::*;` (and
similar for `alloc`/`core`) to try and load the pre-built libraries from the
sysroot (this is the same approach as packages like [embed-rs][embed-rs-source]
take today, using `path` dependencies for the standard library to shadow it).

↩ [*Registries*][registries]

### Why can `builtin_deps` shadow other packages in the registry?
[rationale-cargo-index-shadowing]: #why-can-builtin_deps-shadow-other-packages-in-the-registry

While `crates.io` forbids certain crate names including `std`, `alloc` and
`core`, third party registries may allow it without a warning. The schema needs
a way to refer to packages with the same name either in the registry or builtin,
which `builtin_deps` allows.

`builtin_deps` names are not allowed to shadow names of packages in `deps` as
these would conflict when passed to rustc via `--extern`.

↩ [*Registries*][registries]

## Rebuilding the standard library
[rationale-rebuilding-the-standard-library]: #rebuilding-the-standard-library-1

These rationale/alternatives apply to the content in the
[*Rebuilding the standard library*][rebuilding-the-standard-library] section.

### Why put `build-std` in the Cargo config?
[rationale-build-std-in-config]: #why-put-build-std-in-the-cargo-config

The main [motiviations][motivation] to rebuild the standard library inherently
come from the target platform of the project or the codegen flags used during
compilation. This means the author of the root project may want a higher level
of control than filtering on the target triple or `cfg` options in the
`Cargo.toml` would allow.

The Cargo configuration does not aggregrate based on dependencies. This
behaviour is not really required as each library would likely come to the same
conclusion about whether the user wants to enable build-std and would probably
necessitate a top-level user override anyway in case a library guessed
incorrectly.

It also seems to be true that the standard library features currently available
should also be controlled by the top-level user rather than set by dependencies
and resolved together, which suggests that Cargo features may not be the correct
mechanism for configuring the standard library.

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

### Why accept `off` as a value for `build-std`?
[rationale-build-std-off]: #why-accept-off-as-a-value-for-build-std

While not a default value, the user can specify `off` if they prefer which will
never rebuild the standard library. rustc will still return an error when the
user's target-modifiers do not match the prebuilt standard library.

The `off` value is useful particularly for qualified toolchains where rebuilding
the standard library may invalidate the testing that the qualified toolchain has
undergone.

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

### Why not always rebuild when the profile changes?
[rationale-why-not-always-rebuild]: #why-not-always-rebuild-when-the-profile-changes

Cargo's users don't currently expect that changing any part of their profile
configuration, such as trying a different optimisation level, would trigger a
rebuild of the standard library. For small projects, rebuilding the standard
library could be a significant increase in the overall build time for a project.

If `build-std = "always"` were the default, the standard library could be
rebuilt quite frequently without much benefit. Especially as the pre-built
standard library is built using the release profile, all debug profile builds
would immediately trigger a rebuild of the standard library.

See
[*Why not ship a debug profile `rust-std?*][why-not-ship-a-debug-profile-rust-std]
and
[*Why rebuild the standard library automatically?*][rationale-why-automatic]

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

### Why have different build-std defaults depending on the profile?
[rationale-different-defaults]: #why-have-different-build-std-defaults-depending-on-the-profile

`build-std = "target-modifiers"` is intended to minimise the incidences of
rebuilding the standard library, which is desirable when doing local
development. This corresponds to the typical use case of Cargo's `dev` and
`test` profiles.

`build-std = "match-profile"` is intended to be used when additional time spent
building the standard library is not a problem and the quality of the final
artifact is paramount. This corresponds closely with the release profile, where
additional time spent on optimisations (e.g. with `-Ctarget-cpu`) is acceptable.
Always re-building the standard library with the user's profile configuration in
release mode is likely to result in a more optimised build than with the
pre-built standard library and is thus a reasonable default for the `release`
and `bench` profiles.

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

### Why rebuild the standard library automatically?
[rationale-why-automatic]: #why-rebuild-the-standard-library-automatically

There are a variety of alternatives to rebuilding the standard library
automatically:

1. Cargo could continue to use an explicit command-line flag to enable
   build-std, such as the current `-Zbuild-std` (stabilised as `--build-std`).

   This approach is proven to work, as per the current unstable implementation,
   but has a poor user experience, requiring an extra argument to every
   invocation of Cargo with almost every subcommand of Cargo.

   However, this approach does not lend itself to use with other future and
   current Cargo features. Additional flags would be required to enable Cargo
   features (like today's `-Zbuild-std-features`) and would still necessarily be
   less fine-grained than being able to enable features on individual standard
   library crates. Similarly for public/private dependencies or customising the
   profile for the standard library crates.

2. Cargo could automatically rebuild the standard library as proposed in this
   RFC but without declaring dependencies on the standard library at all in
   `Cargo.toml`. See
   [*Why explicitly declare dependencies on the standard library in `Cargo.toml`?*][rationale-why-explicit-deps].
   This is similar to the approach taken by [rfcs#2663][rfcs-2663-2019].

3. Cargo could have a global opt-in for rebuilding the standard library in the
   Cargo configuration.

   This approach could work well but could only be configured globally or on a
   per-target basis, rather than on a per-project basis for only those projects
   that need build-std. Furthermore, users would need to learn about this
   configuration option and enable it when they encounter circumstances which
   require build-std (like enabling a target modifier), which increases the
   cognitive load of those features.

   It is similar to the approach proposed by
   [cargo#4959][xargo-and-cargo-4959-2016].

This proposal prefers automatic rebuilding of the standard library to the above
alternatives. Rebuilds of the standard library happening transparently reduce
the requirement that users learn about build-std as something to enable and
configure. Combined with explicit dependencies on the standard library crates,
build-std can avoid any cost on users that do not require it (by triggering
automatically when a target modifier is changed, and having no unnecessary
rebuilds otherwise).

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

### Why use the lockfile of the `rust-src` component?
[rationale-lockfile]: #why-use-the-lockfile-of-the-rust-src-component

Using different dependency versions for the standard library would invalidate
the upstream testing of the standard library guaranteeing that the standard
library works as expected for a target, per the target tier policy. In
particular, some crates use unstable APIs when included as a dependency of the
standard library meaning that there is a high risk of breakage if any package
version is changed.

Using the lockfile included in the `rust-src` component guarantees that the same
dependency versions are used as in the pre-built standard library. As the
standard library `crates.io` dependencies are private, it does not re-export
types from its dependencies, this will not affect interoperability with the
same dependencies of different versions used by the user's crate.

Using the lockfile does prevent Cargo from resolving the standard library
dependencies to newer patch versions that may contain security fixes. However,
this is already impossible with the prebuilt standard library.

See
[*Why vendor the standard library's dependencies?*][why-vendor-the-standard-librarys-dependencies]

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

### Why not build the standard library in incremental?
[rationale-incremental]: #why-not-build-the-standard-library-in-incremental

As the standard library sources are never modified, incremental compilation
would only add a compilation time overhead.

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

### Why not produce a `dylib` for the standard library?
[rationale-no-dylib]: #why-not-produce-a-dylib-for-the-standard-library

The `std` crate's `Cargo.toml` is configured with
`crate-type = ["rlib", "dylib"]` so it can produce both artifacts. The Rust
project ships both artifacts, with the `dylib` only linked against when
`-Cprefer-dynamic` is enabled. However, the `dylib` is not part of Rust's
stability guarantee so a first-class way of specifying crate types is left to a
future extension.

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

### Why use the pre-built standard library for procedural macros and build-scripts?
[rationale-sysroot-for-host-deps]: #why-use-the-pre-built-standard-library-for-procedural-macros-and-build-scripts

Procedural macros and build scripts always run on the host and need to be built
with a configuration that are compatible with the host toolchain's Cargo and
rustc. There is little advantage to using a custom standard library with
procedural macros or build scripts, as they are not part of the final output
artifact and anywhere they can run already have a toolchain with host tools and
a pre-built standard library. Procedural macros must link against the compiler
which further limits potential use cases to those without `target-modifiers`.

See also
[*Why not support implicit or explicit standard library dependencies in `build-dependencies`?*][rationale-no-deps-in-build-deps]

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

### Why default to assuming the pre-built standard library is the release profile?
[rationale-assume-release-profile]: #why-default-to-assuming-the-pre-built-standard-library-is-the-release-profile

The pre-built standard library is built in the release profile, with minor
overrides to the default profile set in the standard library's workspace. As
Cargo will be able to read and observe the definition of this profile in the
standard library and its workspace's `Cargo.toml`, assuming that the pre-built
artifact matches this is a lightweight and best-effort mechanism to determine
when a rebuild of the standard library is necessary.

This mechanism is unfortunately somewhat fuzzy and is a lightweight and
best-effort mechanism of avoiding unnecessary rebuilds of the standard library.
Alternatively, rustc could expose the ability to dump the configuration options
used with an rlib so that Cargo could compare against the compilation flags it
intends to use, or rustc could take an rlib and the proposed compilation flags
as arguments and return an exit code indicating whether the provided flags
differ from those used with the rlib.

↩ [*Profiles*][profiles]

### Why not ship a debug profile `rust-std`?
[rationale-ship-debug-std]: #why-not-ship-a-debug-profile-rust-std

As the default configuration for `build-std` is `target-modifiers`, debug builds
of the user's crate would not trigger a rebuild of the standard library and
would use the pre-built standard library (as the default release profile does
not change any target modifiers compared to the default debug profile). It would
only be when `build-std = "always"` that any debug build would first trigger a
rebuild of the standard library.

To improve the user experience in this circumstance, it could be worth shipping
a debug profile `rust-std`, but as this is not the common case, it isn't
proposed in this RFC. Some intrinsics rely on optimisations so a debug profile
standard library may result in counterintuitive or unexpected behaviour for
users. If a debug `rust-std` was eventually made available, it might be expected
that it be used for any `debug` profile build, which would involve more
machinery.

↩ [*Profiles*][profiles]

### Why respect the profile of the standard library workspace?
[rationale-respect-std-profile]: #why-respect-the-profile-of-the-standard-library-workspace

Profiles provide a useful place to declaratively write the configuration used
for the pre-built standard library in such a way that Cargo can read it during
build-std. By contrast, configuration for the pre-built standard library that is
defined in bootstrap is entirely opaque to Cargo.

While it is a divergence from other dependencies to respect the profile of the
standard library crates, when the standard library is rebuilt, the newly-built
standard library will deviate as little as possible from the pre-built standard
library.

↩ [*Profiles*][profiles]

### Why merge the user's profile and the standard library workspace's profile?
[rationale-why-merge]: #why-merge-the-users-profile-and-the-standard-library-workspaces-profile

It is still desirable for the user's profile to apply to the standard library
and for this to work transparently, so that the user does not need to
specifically write profile overrides for the standard library crates in order to
influence its build - customising the build of standard library dependencies can
work like customising the build of any other dependency.

User profile changes only apply to the standard library when they have changed
from their default option values so as to keep the standard library build as
close to the pre-built standard library as possible. Profile changes are merged
to preserve necessary defaults from the standard library's profiles.

↩ [*Profiles*][profiles]

### Why respect profile overrides of the standard library's workspace?
[rationale-respect-profile-overrides]: #why-respect-profile-overrides-of-the-standard-librarys-workspace

Respecting the profile overrides in the standard library's workspace will ensure
that compiler-builtins' profile overrides continue to apply and the crate will
be built with a large number of codegen units to force each intrinsic into its
own CGU and be deduplicated with `libgcc`.

↩ [*Profiles*][profiles]

### Why not allow profile overrides to override the standard library's dependencies?
[rationale-why-not-override-std-deps]: #why-not-allow-profile-overrides-to-override-the-standard-librarys-dependencies

The dependencies of the standard library are an implementation detail and ought
not be observable to the user.

↩ [*Profiles*][profiles]

### Why prevent rustc from loading root dependencies from the sysroot?
[rationale-root-sysroot-deps]: #why-prevent-rustc-from-loading-root-dependencies-from-the-sysroot

Loading root dependencies from the sysroot could be a source of bugs.

For example, if a crate has an explicit dependency on `core` which is newly
built, then there will be no `alloc` or `std` builds present. A user could still
write `extern crate alloc` and accidentally load `alloc` from the sysroot
(compiled with the default profile settings) and consequently `core` from the
sysroot, conflicting with the newly build `core`. `extern crate alloc` should
only be able to load the `alloc` crate if the crate depends on it in its
`Cargo.toml`. A similar circumstance can occur with dependencies like
`panic_unwind` that the compiler tries to load itself.

Dependencies of packages can still be loaded from the sysroot, even with
`--no-implicit-sysroot-deps`, to support the circumstance where Cargo uses a
pre-built standard library crate (e.g.
`$sysroot/lib/rustlib/$target/lib/std.rlib`) and needs to load the dependencies
of that crate which are also in the sysroot.

`--no-implicit-sysroot-deps` is a flag rather than default behaviour to preserve
rustc's usability when invoked outside of Cargo. For example, by compiler
developers when working on rustc.

`--sysroot=''` is an existing mechanism for disabling the sysroot - this is not
used as it remains desirable to load dependencies from the sysroot as a
fallback. In addition, rustc uses the sysroot path to find `rust-lld` and
similar tools and would not be able to do so if the sysroot were disabled by
providing an empty path.

↩ [*Preventing implicit sysroot dependencies*][preventing-implicit-sysroot-dependencies]

### Why use `noprelude` with `--extern`?
[rationale-noprelude-with-extern]: #why-use-noprelude-with---extern

The `noprelude` modifier for `--extern` is necessary for use of the `--extern`
flag to be equivalent to using a modified sysroot.

Without `noprelude`, rustc implicitly inserts a `extern crate $name` when using
`--extern`. As a consequence, if a newly-built `alloc` were passed using
`--extern alloc=alloc.rlib` then `extern crate alloc` would not be required, but
it would be if the pre-built `alloc` could be used. This difference in how a
crate is made available to rustc should not be observable to the user.

↩ [*Preventing implicit sysroot dependencies*][preventing-implicit-sysroot-dependencies]

### Why not allow the source path for the standard library be customised?
[rationale-custom-src-path]: #why-not-allow-the-source-path-for-the-standard-library-be-customised

It is not a goal of this proposal to enable or improve the usability of custom
or modified standard libraries.

↩ [*Vendored `rust-src`*][vendored-rust-src]

### Why vendor the standard library's dependencies?
[rationale-vendoring]: #why-vendor-the-standard-librarys-dependencies

Vendoring the standard library's dependencies has multiple advantages..

- Avoid needing to support standard library dependencies in `cargo vendor`
- Avoid needing to support standard library dependencies in `cargo fetch`
- Re-building the standard library does not require an internet connection
- Standard library dependency versions are fixed to those in the `Cargo.lock`
  anyway, so initial builds with `build-std` start quicker with these
  dependencies already available
- Allow build-std to continue functioning if a `crates.io` dependency is
  "yanked"
  - This leaves the consequences of a toolchain version using yanked
    dependencies the same as without this RFC

..and few disadvantages:

- A larger `rust-src` component takes up more disk space and takes longer to
  download
  - If using build-std, these dependencies would have to be downloaded at build
    time, so this is only an issue if build-std is not used and `rust-src` is
    downloaded.
- Vendored dependencies can't be updated with the latest security fixes
  - This is no different than the pre-built standard library

How this affects `crates.io`/`rustup` bandwidth usage or user time spent
downloading these crates is unclear and depends on user patterns. If not
vendored, Cargo will "lazily" download them the first time `build-std` is used
but this may happen multiple times if they are cleaned from its cache without
upgrading the toolchain version.

See
[*Why use the lockfile of the `rust-src` component?*][why-use-the-lockfile-of-the-rust-src-component]

↩ [*Vendored `rust-src`*][vendored-rust-src]

### Why not check if `rust-src` has been modified?
[rationale-src-modifications]: #why-not-check-if-rust-src-has-been-modified

It is likely that any protections implemented to check that the sources in
`rust-src` have not been modified could be trivially bypassed.

Any crate that depends on `rust-src` having been modified would not be usable
when published to crates.io as the required modifications will obviously not be
included.

↩ [*Vendored `rust-src`*][vendored-rust-src]

### Why allow building from the sysroot with implied `RUSTC_BOOTSTRAP`?
[rationale-implied-bootstrap]: #why-allow-building-from-the-sysroot-with-implied-rustc_bootstrap

Cargo needs to be able to build the standard library crates, which inherently
require a nightly toolchain. It could set `RUSTC_BOOTSTRAP` internally to do
this with a stable toolchain, however this is a shared requirement with other
build systems that wish to build an unmodified standard library and want to work
on stable toolchains.

For example, Rust's project goal to enable Rust for Linux to build using only a
stable toolchain would require that it be possible to build `core` without
nightly.

It is not sufficient for rustc to special-case the `core`, `alloc` and `std`
crate names as when being built as part of the standard library, dependencies of
the standard library also use unstable features and so these crate would also
need such special-casing, which is not practical.

↩ [*Building the standard library on a stable toolchain*][building-the-standard-library-on-a-stable-toolchain]

### Why not use weak linkage for `compiler-builtins/mem` symbols?
[rationale-no-weak-linkage]: #why-not-use-weak-linkage-for-compiler-builtinsmem-symbols

Since [compiler-builtins#411], the relevant symbols in `compiler_builtins`
already have weak linkage. However, it is nevertheless not possible to simply
remove the `mem` feature and have the symbols always be present.

Some targets, such as those based on MinGW, do not have sufficient support for
weak definitions (at least with the default linker). Furthermore, weak linkage
has precedence over shared libraries and the symbols of a dynamically-linked
`libc` should be preferred over `compiler_builtins`'s symbols.

↩ [*`compiler-builtins-mem`*][compiler-builtins-mem]

### Why permit breakage of nightly build-std users using tier three targets?
[rationale-breakage]: #why-permit-breakage-of-nightly-build-std-users-using-tier-three-targets

As this use case would previously have been using the unstable
`-Zbuild-std=core`, this user must be on a nightly toolchain. This proposal
argues that this breakage is unfortunate but acceptable, as it does not impact
users on a stable toolchain, and once the `no_std` ecosystem has updated its
dependencies on the standard library, these users will no longer need to rely on
any nightly features to build the standard library (if this iteration of
build-std is eventually stabilised).

Alternatively, the `-Zbuild-std` flag could remain temporarily which would
override the explicit dependencies on the standard library declared on the
user's crate graph. This could be used to avoid breakage by users who are
currently using `-Zbuild-std` successfully and whose crate graph has not yet
updated to explicitly declare their dependencies on `core`. As explicitly
declaring dependencies on `core` would permit use of a stable toolchain, users
would have an incentive to do this and move off of the `-Zbuild-std` flag over
time.

↩ [*Potential migration breakage*][potential-migration-breakage]

### Why not globally cache builds of the standard library?
[rationale-caching]: #why-not-globally-cache-builds-of-the-standard-library

The standard library is no different than regular dependencies in being able to
benefit from global caching of dependency builds. A generic proposal for global
dependency caching could support the standard library. It is out-of-scope of
this proposal to propose a special-cased mechanism for this that applies only to
the standard library.

↩ [*Caching*][caching]

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The following small details are likely to be bikeshed prior to RFC acceptance or
stabilisation and aren't pertinent to the overall design:

## What syntax is used to identify dependencies on the standard library in `Cargo.toml`?
[unresolved-dep-syntax]: #what-syntax-is-used-to-identify-dependencies-on-the-standard-library-in-cargotoml

What syntax should be used for the explicit standard library dependencies?
`builtin = true`? `sysroot = true`? `version = "*"`?

↩ [*Standard library dependencies*][standard-library-dependencies]

## What is the format for builtin dependencies in `Cargo.lock`?
[unresolved-lockfile]: #what-is-the-format-for-builtin-dependencies-in-cargolock

How should `builtin` deps be represented in lockfiles? Is `builtin = true`
appropriate? Could the `source` field be reused with the string "builtin" or
should it stay only as a URL+scheme?

↩ [*Standard library dependencies*][standard-library-dependencies]
  
## What syntax is used to patch dependencies on the standard library in `Cargo.toml`?
[unresolved-patch-syntax]: #what-syntax-is-used-to-patch-dependencies-on-the-standard-library-in-cargotoml

`[patch.builtin]` is the natural syntax given `builtin` is a new source, but may
be needlessly different to existing packages.

↩ [*Patches*][patches]

## Where should the `build-std` configuration in `.cargo/config` be and what should it be called?
[unresolved-config-location-name]: #where-should-the-build-std-configuration-in-cargoconfig-be-and-what-should-it-be-called

Should it be a top-level key or in a section? What should it be called?
`build-std`? `rebuild-standard-library`?

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

## What should the values of the `build-std` config be named?
[unresolved-config-values]: #what-should-the-values-of-the-build-std-config-be-named

What is the most intuitive name for the values of the `build-std` setting?
`always`? `match-profile`? `rebuild-builtins`?

↩ [*Rebuilding the standard library*][rebuilding-the-standard-library]

# Future possibilities
[future-possibilities]: #future-possibilities

There are many possible follow-ups to build-std.

## Allow custom targets with build-std
[future-custom-targets]: #allow-custom-targets-with-build-std

This would require a decision from the relevant teams on the exact stability
guarantees of the target-spec-json format and whether any large changes to
the format are desirable prior to broader use.

↩ [*Custom targets*][custom-targets]

## Allow enabling/disabling features with build-std
[future-features]: #allow-enablingdisabling-features-with-build-std

This would require the library team be comfortable with the features declared on
the standard library being part of the stable interface of the standard library.

The behaviour of disabling default features has been highlighted as a potential
cause of breaking changes.

Alternatively, this could be enabled alongside another proposal which would
allow the standard library to define some features as stable and others as
unstable.

↩ [*Features*][features]

## Avoid building `panic_unwind` unnecessarily
[future-panic_unwind]: #avoid-building-panic_unwind-unnecessarily

This would require adding a `--print default-unwind-strategy` flag to rustc and
using that to avoid building `panic_unwind` if the default is abort for any
given target and `panic` is not set in the profile.

↩ [*Panic strategies*][panic-strategies]

## Enable local recompilation of special object files/sanitizer runtimes
[future-recompile-special]: #enable-local-recompilation-of-special-object-filessanitizer-runtimes

These files are shipped pre-compiled for relevant targets and are not compiled
locally. If a user wishes to customise the compilation of these files like the
standard library, then there is no mechanism to do so.

↩ [*Special object files*][special-object-files]

# Appendix I: Summary of features to be implemented
[appendix-i]: #appendix-i-summary-of-features-to-be-implemented

There are many features proposed in this RFC for different parts of the project:

- Compiler
  - [`--no-implicit-sysroot-deps`][preventing-implicit-sysroot-dependencies]
  - [Loading `panic_unwind` from `-L dependency=`][rebuilding-the-standard-library]
  - [`--print target-standard-library-support`][target-standard-library-support]
- Cargo
  - [Standard library dependencies][standard-library-dependencies]
  - [Rebuilding the standard library][rebuilding-the-standard-library]
- Standard library
  - [Moving configuration into the standard library's profile][profiles]

# Appendix II: Exhaustive literature review
[appendix-ii]: #appendix-ii-exhaustive-literature-review

This section will attempt to summarize every issue, pull request, RFC and
discussion related to the design and implementation of build-std since its
conception in May 2015. If anything has been omitted then that's just an
oversight and it can be added. The summaries may not reflect current up-to-date
information if those updates weren't in the discussion being summarized.
Up-to-date information should be present when these issues are referenced in the
previous sections.

This section's level of detail is not strictly necessary to understand this RFC,
the summary at the start of the [History][history] section should be
sufficient, but this section should help if more detail is desired on some
referenced material.

## [rfcs#1133] (2015)
[a1-rfcs-1133-2015]: #rfcs1133-2015-1

This section contains all of the sources related to [rfcs#1133].

- **[rfcs#1133]: Make Cargo aware of standard library dependencies**,
  [Ericson2314], May 2015
  - This is the first RFC that proposed making Cargo aware of the standard
    library
  - It was motivated by..
    - ..better support for cross-compilation to targets that cannot have
      pre-built std due to strange configuration requirements
    - ..building std with different configurations (e.g. panic
      strategies/features/etc)
    - ..simplifying `rustbuild`
  - The RFC proposes both that the standard library should be explicitly listed
    as a dependency in `Cargo.toml` and that it should be rebuilt when necessary
    - `std = { version = "1.10", stdlib = true }` is the proposed syntax for a
      `std` dependency in `Cargo.toml`
      - `version = "*"` is also acceptable. As today, a package specifying a
        version for the standard library will be rejected by crates.io
      - `stdlib = true` defines the source of the dependency (similarly to `git`
        or `path`)
      - As the compiler/language version is tied to the standard library
        version, this is effectively declaring an MSRV
      - This dependency would be implicitly added to all crates unless a
        standard library dependency is explicitly written
          - i.e. writing `core = ..` would prevent the default `std = ..`
            dependency from being added implicitly
          - An `implicit-dependencies` key would also be added and to determine
            whether implicit standard library dependencies are added. This is to
            be able to write the `Cargo.toml` of the `core` crate
          - `no_std` was not stable at the time of this RFC being written, so
            the RFC does not address the circumstance where an implicit
            dependency may make an existing `no_std` crate fail to build for a
            tier three target that does not support the standard library
    - The RFC is written prior to the introduction of the `rust-src` component
      and assumes that other implementations of the compiler may exist that will
      have their own standard library implementations and that the feature must
      support that
    - A description of how the feature could be implemented in Cargo is
      provided:
      - Add implicit dependencies as early as possible so the majority of Cargo
        requires fewer changes
      - `stdlib = true` generates a "source id" mapping to either a compiler
        source directory (e.g. `rust-src` today) or a "sysroot binary mock
        source"
        - A "sysroot binary mock source" is the source containing the pre-built
          standard library and is found by searching the sysroot for the
          standard library rlib
      - It always builds the source if present, falling back to the sysroot
        rlibs otherwise
        - The RFC argues that as the configuration of the standard library to
          build is not known, the implementation must be conservative (e.g. no
          features are enabled)
      - It is intended that rustc would never use the sysroot (by passing
        `--sysroot=''`)
      - The standard library would not be added to lockfiles, the implementation
        argues that there would be no value in adding it
    - The RFC aims to be forward-compatible with the standard library source
      being replaced by users and that parts of the standard library could be on
      crates.io
    - The RFC also suggests adding a mechanism for this to be introduced to
      Cargo as an unstable feature, as Cargo had no mechanism for this when the
      RFC was written
  - There was varied feedback over a period of three years, initially:
    - The advantage of replacing the standard library was not clear, as
      `std = { path = "..." }` is already possible with Cargo (and used by some
      projects to wrap the standard library)
      - This doesn't work if a user wants to replace `std` over the entire
        dependency graph
    - Declaring a standard library dependency explicitly in all crates was seen
      as unfortunate by some as it was redundant in many cases
  - [xargo] was mentioned for the first time in May 2016.
  - [cargo#2768] was opened as an initial implementation of the RFC.
  - After core/std was being built using bootstrap and Cargo, there was another
    burst of activity on the RFC:
    - Supporting `compiler-rt` was raised as a challenge, but it was argued that
      this wasn't a blocker as they are not necessary for the very bare metal
      use-case intended by the author
    - Lots of possibilities that the RFC enabled were raised on the discussion
      thread
    - There was discussion about what would trigger a build of std versus re-use
      of std from the sysroot:
        - The RFC wasn't exceptionally clear on this, but it appears that the
          intent was for the `Cargo.toml` to be the source-of-truth for what a
          package needs, and that the environment/profile settings determine
          whether a build needs to happen
        - Cargo would pass `--sysroot-whitelist` with a list of crate names to
          the compiler when it determines that rustc should be permitted to use
          the pre-built artifacts from the sysroot
    - It was repeatedly expressed by the author and advocates of the RFC that a
      ideal final state would be that the sysroot as a concept not exist (at
      least for loading dependencies)
    - Contemporary project members discussed the RFC and shared their
      conclusions:
      - Sysroot crates being specified with a special syntax is reasonable
      - Putting std on crates.io is a non-goal
      - The sysroot exists and won't go away anytime soon
      - A known location in the sysroot for the std sources is desirable/good
      - The std facade should be a normal part of the crate DAG
        - This was not true when the comment was written in July 2016
      - It should be possible to override features of the sysroot crates
      - Features should be used to avoid C dependencies
      - Sysroot replacement poses stabililty concerns so must be restricted to
        nightly
  - By August 2016, the participating project members seemed quite happy with
    the RFC and shared some more feedback:
    - There was interest in requiring semver versions for the standard library,
      as a pseudo-MSRV
    - `build-dependencies` and `dev-dependencies` shouldn't have explicit
      standard library dependencies
    - Didn't want a `--no-resolve-sysroot` flag (to prevent loading crates from
      the sysroot) as this would break people who use unstable crates from the
      sysroot
    - There were specific implications identified with respect to
      versioning/dependency specification:
      - Crates in the sysroot would need to have the same version as the
        compiler
      - Standard library dependencies effectively allow language version pinning
      - A `^1.0` version would add constraints on Rust 2
      - It would stabilise the name of `test`
      - A mechanism for building unstable code in the sysroot on the stable
        compiler would be necessary
        - This is the current use case desired by Rust for Linux
        - There were many concerns about avoiding abuse of this and discussion
          of mechanisms to confirm that only the intended unstable crates were
          built on a stable compiler
  - The RFC stalled for a few months and then was blocked by a mechanism for
    unstable Cargo features in July 2017
  - In February 2018, the RFC was closed in favour of [cargo#4959]
- **[cargo#2768]: Phase 1 of stdlib dependencies**, [Ericson2314], Jun 2016
  - An initial implementation of [rfcs#1133].
  - This PR received almost no feedback.
- **[cargo#5002]: Explicit standard library dependencies**, [Ericson2314], Feb
  2018
  - A issue split-off from [rfcs#1133] with the goal of discussing how explicit
    dependencies on the standard library could work
  - This issue received no feedback
- **[cargo#5003]: Implicit standard library dependencies**, [Ericson2314], Feb
  2018
  - A issue split-off from [rfcs#1133] with the goal of discussing how implicit
    dependencies on the standard library could work
  - This issue received no feedback
  
## [xargo] and [cargo#4959] (2016)
[a1-xargo-and-cargo-4959-2016]: #xargo-and-cargo4959-2016-1

This section contains all of the sources related to [xargo] and [cargo#4959]:

- **[xargo]**, [japaric], Mar 2016
  - A developer tool that wraps Cargo to provide build-std-like functionality
    - Built while [rfcs#1133] was still being discussed
  - Builds custom sysroots based on the configuration in a `Xargo.toml`
    - Requires nightly as it builds the standard library and gets standard
      library sources from the `rust-src` component
    - Supports multi-stage builds, features, flags, etc.
  - Xargo is used instead of Cargo and has the same command line interface, but
    ensures that the locally-built sysroots are used
  - In maintenance mode since January 2018
    - Cargo team were interested in incorporating its functionality at the time
- **[cargo#4959]: Sysroot building functionality**, [japaric], Jan 2018
  - Successor of [rfcs#1133], inspired by [xargo]
  - Xargo was fairly widely used but had issues due to being a wrapper around
    Cargo rather than a part of Cargo
    - It didn't replicate Cargo's fingerprinting so would unnecessarily perform
      rebuilds
    - It didn't track changes to sysroot sources to trigger rebuilds when they
      changed
    - It didn't have a rustup shim so users couldn't use the toolchain selection
      arguments typical of rustup shims (e.g. `+nightly`)
    - There were deviations from built-in Cargo commands
  - It suggested adding a `[sysroot]` section in the `.cargo/config`
    - Could specify features/flags/stages for
      `core`/`compiler_builtins`/`alloc`/`std`/etc
    - Can also specify `rust-src` path
      - Optional, defaults to `rust-src` component path
    - If set, Cargo would rebuild sysroot crates for targets, put them in the
      `target` directory and use them for rustc/rustdoc/etc
      - These would be rebuilt when profiles change, using Cargo's usual
        fingerprinting mechanism
      - These crates would always be built using the release profile
    - Like xargo, would support multi-stage builds
      - e.g. `std` built with default sysroot, then `test` built in sysroot w/
        new `std`, then a new sysroot made with both the new `test` and new
        `std`
    - None of it would apply to `build.rs`
  - There was a variety of feedback:
    - There should be restrictions on what crates could be put in the sysroot
      should be added so that this mechanism could not be used for third-party
      crates
    - Procedural macros would also need to use the host sysroot
    - Syntax was not user-friendly enough
    - The ability to customise std and not require the `rust-src` component be
      used was requested
      - Similarly, requests for `[patch]` sections
    - There was a mention of a immiment merging of rustup and Cargo which could
      be relevant
      - Spoiler: this didn't happen
    - Exposing bootstrap stages is cumbersome, error-prone and brittle and Cargo
      should just figure that part out automatically
    - There were arguments against further reliance on the sysroot as a concept
      - Incremental/IDE support is allegedly more challenging with the sysroot
      - Some proposed features in the ether circa March 2018 related to
        module-system namespacing/ligher `extern crate` would be made more
        challenging if the entire sysroot were automatically imported
      - Many of the other arguments against the sysroot were actually arguments
        against a pre-built standard library that cannot possibly satify all
        users
- **[cargo-xbuild]**, [rust-osdev], May 2018
  - A simplified fork of `xargo`
  - Wrapper around `cargo build` that uses a custom sysroot according to the
    configuration in `package.metadata.cargo-xbuild` in `Cargo.toml`
  - Now recommends using `--build-std` in Cargo instead

## [rfcs#2663] (2019)
[a1-rfcs-2663-2019]: #rfcs2663-2019-1

This section contains all of the sources related to [rfcs#2663]:

- **[rfcs#2663]: `std` Aware Cargo**, [jamesmunns], Mar 2019
  - Re-building the standard library in this proposal is motivated by:
    - Being able to use the standard library with tier three and custom targets
      that do not have a pre-built standard library
    - Customising the standard library with different feature flags
    - Applying different codegen flags to the standard library
  - This proposal largely focuses on what the user experience of build-std
    should be and has many unresolved questions and details left for the
    implementors to work out
  - Unlike [rfcs#1133], this RFC only focuses on when the standard library
    should be rebuilt, rather than how a dependency on the standard library
    should be declared
  - It is primarily inspired by [xargo] and also cites [rfcs#1133],
    [cargo#5002] and [cargo#5003]
  - There are four objectives of the RFC:
    - Allow core to be recompiled when a target isn't available from rustup
      - This should be possible on a stable toolchain, but will require nightly
        to..
        - ..set feature flags
        - ..modify core
      - Cargo should recompile core when a custom target spec is used, feature
        flags are modified, profile configuration is changed or the sysroot is
        patched
          - As above, on a stable toolchain, only the profile configuration will
            be able to be changed to trigger a rebuild
          - Only the root crate's feature flags and profiles would be respected,
            not those of dependencies
      - Unless opting into Cargo features, core does not need to be explicitly
        specified as a dependency in `Cargo.toml`
      - Sources for core are from the `rust-src` component
      - The proposal lists three implications of the above feature:
        - Target specification format would need to be stabilised
          - It isn't explained why this is a necessary implication
        - Rust implementation of `compiler_builtins` would need to be used for
          custom targets
        - `RUSTC_BOOTSTRAP` needs to be set for core
    - Allow use of "stable" Cargo features from Cargo
      - This section proposes that a mechanism exist to declare Cargo features
        as stable and unstable and that only stable features be usable on the
        stable channel (i.e. resolving [cargo#10881])
        - There are no specifics on how these features would be declared, but
          the syntax for using a feature is described as being identical for
          stable and unstable features
      - Unstable feature flags in the standard library could be stabilised by
        following the same stabilisation process as anything else in the
        standard library
      - The proposal does not mention anything about the unification of the
        features of the dependencies of the standard with those same
        dependencies of the user's crate
    - Allow alloc/std to be recompiled when a target isn't available and allow
      use of "stable" Cargo features from alloc/std
      - Exactly as above
      - Doesn't address complexities with alloc/std - when does Cargo need to
        build alloc or std and not just core? How does Cargo know this?
    - Allow users to provide their own versions of core/alloc/std
      - Would always require nightly
      - Suggests using regular patch syntax under `patch.sysroot` key
  - There are many unresolved questions in the RFC:
    - How to specify a dependency on the standard library in a crate?
    - How does a `no_std` crate tell Cargo it doesn't need to build std?
    - Should std be rebuilt if core is?
    - Should there be tamper-detection for the standard library?
    - Should the standard library be built locally (per project) or globally
      (shared between projects)?
    - What to do with the standard library's `Cargo.lock`?
    - Should profile overrides *always* rebuild std?
    - Should providing a custom standard library require nightly?
    - Should customising the standard library be permitted?
  - There are also a handful of future possibilities mentioned:
    - core/std could be unified into a single crate with different features to
      represent the differences that exist between the crates
    - The project could choose to stop shipping a precompiled standard library
  - This RFC was sufficiently vague that it was unlikely to be accepted as
    written - to accept it is largely to assert that something like build-std is
    desired, which is not controversial, but the tricky details which make
    build-std difficult would still need to be designed and discussed
  - A variety of feedback was received:
    - Enabling customisation of std may not be practical because it would
      require all combinations of Cargo feature flags be tested
      - It is asserted that this testing would be necessary without considering
        alternatives (such as a documented stability policy for the standard
        library's feature flags)
      - It is suggested that only feature flags of omission (that remove parts
        of the standard library) be used to avoid this
        - It was later relayed that Cargo features do not work this way
      - Another suggestion was only no flags enabled and all flags enabled could
        be tested and that this would be sufficient
    - There is disagreement over whether specifying a custom standard library
      should require nightly
      - It is already possible to specify a standard library dependency in
        `Cargo.toml` with a path and have it override that crate
        - At the time of the conversation, this was used by the `embed-rs` crate
          to override some of the standard library's future types
          ([source][embed-rs-source]/[`Cargo.toml`][embed-rs-cargo-toml])
      - Some argue that it should be possible to do this on stable because any
        alternative implementation would necessarily only build on nightly
        anyway
    - It is argued both that the target-spec-json format *would* and *would not*
      need to be stabilised if it were supported by build-std
      - As build-std would make it more practical to use these custom targets on
        a stable toolchain, some argue that the format would need to be made
        stable
    - The host sysroot would need to be used for `build.rs` and procedural
      macros
    - There are C dependencies of the standard library that are difficult to
      build and would need some consideration of this to ensure the usability of
      build-std
      - `libbacktrace` has been replaced by `gimli` so this is less of a concern
        now
    - There are concerns that rebuilding the standard library whenever the
      profile is changed would be too disruptive
        - e.g. changing the optimisation level triggering a rebuilding of std
          would add to compilation times
        - It is suggested that the standard library only be rebuilt if a crate
          adds an explicit dependency on them
        - Cargo's default profiles may not match the configuration of the
          pre-built standard library
          - Research is required to ensure that the default configuration of the
            standard library is known to Cargo and not just bootstrap
        - Alternatively, only rebuild for ABI-modifying flags
    - Sanitizers require sanitizer runtimes to be present and these are not
      configured by Cargo features
    - Stable/unstable Cargo features is an RFC of its own
    - Only considering the root crate would break dependencies that enable
      specific standard library features
    - Cargo needs to know what crates to build when a rebuild is triggered
      - e.g. only `core`, `alloc` too, `std` as well?
    - There are various arguments that the standard library and its crates
      should not be special-cased in any way
      - There is pushback to this arguing that the standard library is
        inherently special
        - Trying to make the standard library a regular crate that is versioned,
          on crates.io and does not exist in the sysroot is often raised in
          these prior art and makes build-std much more complicated and less
          likely to succeed
    - Users should have the option of using the C implementation of
      `compiler_builtins`
    - Crates will need to be able to specify a lack of dependency on the
      standard library (e.g. `no_std` crates)
    - It is important that a pre-compiled standard library and locally-compiled
      standard library have identical behaviour
  - t-lang [discussed the RFC in a meeting][rfcs#2663-t-lang]
    - There weren't any concerns, except:
      - Niko raised that putting something behind a Cargo feature that was
        previously not gated behind a Cargo feature breaks users who use
        `default-features = false` and this should be resolved as it has
        implications for the standard library's development
        - It was suggested that `default-features = false` could be prohibited
          for standard library dependencies
  - Ultimately closed with interested parties directed to the
    [wg-cargo-std-aware] repository
  - There was additional feedback in the draft version of the RFC which was
    shared with some project members - [jamesmunns/rfcs#1]
    - Initially the RFC did not clarify when Cargo would trigger a rebuild
    - Cargo needs to know how not to do any of the proposed machinery for the
      `core` and `compiler_builtins` crates itself
    - `rustc_inherit_overflow_checks` could be removed
    - Cargo features enabled in the standard library cannot just be decided by
      the root crate
      - There was disagreement about whether this is accurate
    - Interactions with `rustc-std-workspace-core` are unclear
    - `patch.sysroot.$crate` key rather than re-using `patch.$crate` makes the
      standard library special

## [wg-cargo-std-aware] (2019-)
[a1-wg-cargo-std-aware-2019-]: #wg-cargo-std-aware-2019--1

After the above issues, pull requests, RFCs and discussions, the
[wg-cargo-std-aware] was created to host issues related to build-std and the
effort which resulted in the current unstable implementation of build-std.

Unlike prior art which predates [wg-cargo-std-aware], [wg-cargo-std-aware]'s
issues capture the majority of the tricky details involved in build-std but
often do not have much discussion to summarise.

Issues in [wg-cargo-std-aware] are very varied and so are split into eight
categories in this proposal:

### Use cases
[use-cases]: #use-cases

These issues collect and elaborate on the use cases that users have for
build-std:

- **[wg-cargo-std-aware#2]: Build the standard library with custom profile
  settings**, [ehuss], Jul 2019
  - Currently, the standard library is only available with the profile settings
    chosen when Rust is distributed
      - "release" profile with `codegen-units=1` and `opt-level=2`
  - Using different settings is desirable
    - When build-std builds Cargo, it should build using the current profile
    - Profile overrides can be used to use different settings for the standard
      library specifically
  - Users currently do not expect the standard library to be rebuilt when
    changing the profile
    - e.g. setting `opt-level=3` will not currently rebuild std and if it
      started to do so then this could significantly increase build times for
      small projects
    - At time of writing, build-std was intended to be strictly opt-in so this
      would not be an issue
    - It was later clarified that the concern was not "whether configured
      profile settings would apply to the standard library", but rather "whether
      that always triggers a rebuild"
      - build-std should always re-use pre-built artifacts if such artifacts
        exist and match the desired profile
  - Risk that profile overrides could expose internal details about the standard
    library, like the `compiler_builtins` crate
- **[wg-cargo-std-aware#3]: Build the standard library for an unsupported
  target**, [ehuss], Jul 2019
  - build-std is desired to make it easier to build the standard library for
    targets that do not have a pre-built standard library, including unsupported
    targets (i.e. `no_std` environments) and custom targets (using
    target-spec-json)
    - Tools like [xargo] and [cargo-xbuild] are used to do this
    - Custom targets are already supported by Cargo, so work for
      [wg-cargo-std-aware#2] should overlap
  - `no_std` binaries often require nightly features
  - `core` is not maximally portable
  - It may be worth exploring how to lower the barrier to porting the standard
    library to a new platform using build-std
- **[wg-cargo-std-aware#4]: Build the standard library with different cargo
  features**, [ehuss], Jul 2019
  - Some users want to enable or disable Cargo features to remove or modify
    parts of the standard library
    - e.g. to change the formatting machinery to emphasise code size, or remove
      panic machinery when unused
  - There was agreement that this customisation should be limited to code behind
    `cfg(feature = "..")` attributes, not arbitrary `cfg`s
  - [cargo#8490] added `-Zbuild-std-features=` to support this experimentally
  - An example provided was wanting to be able to toggle whether `RefCell` could
    provide backtraces when panicking for outstanding borrows
- **[wg-cargo-std-aware#7]: Custom source standard library**, [ehuss], Jul 2019
  - Some users want to be able to replace the source of the standard library
    with a custom implementation
  - The issue notes that this is unlikely to be stabilised in the foreseeable
    future
  - It was suggested that it would be better to allow the user to supply a
    pre-built artifact for any dependency in the crate graph
  - As of 2022, miri needed its own sysroot build and used [xargo] to do that
- **[wg-cargo-std-aware#19]: Use in rustbuild**, [Ericson2314], Jul 2019
  - build-std could be used as part of the [rust-lang/rust] bootstrap, this
    would..
    - ..simplify bootstrap
    - ..reduce gap between official and user builds
    - ..demonstrate that Cargo is expressive enough
  - Most discussion took place in internals thread
    - See *Dogfooding -Z build-std in rustbuild* below
- **[wg-cargo-std-aware#36]: Better understand the no_std binary use case**,
  [ehuss], Sep 2019
  - `no_std` binaries can be tricky (e.g. may require manually linking libc) and
    it would be nice if this were smoother
  - This use-case seems adjacent to build-std
- **[wg-cargo-std-aware#42]: metaprogramming and bootstrapping**,
  [Ericson2314], Sep 2019
  - `build.rs` and `proc_macro` ought to be able to leverage build-std too
  - There was no further relevant discussion
- **[wg-cargo-std-aware#61]: Can we tailor the compiler-builtins package that is
  compiled and linked togther with the core crate?**, [parraman], Nov 2020
  - User wants to change the `compiler_builtins` version using build-std
  - Closed as duplicate of [wg-cargo-std-aware#7]
- **internals.r-l.o: [Dogfooding -Z build-std in rustbuild][wg-cargo-std-aware#19-internals]**, [Ericson2314], Jan 2021
  - Proposes using `-Zbuild-std` in rustbuild in order to dogfood it to more
    users
  - Suggests that this would lead to Cargo needing to learn some of bootstrap's
    current behaviour and that Cargo would need to be changed more often to
    accomodate changes that bootstrap receives
  - Suggests that changing the Cargo version contained within a beta release
    would make this easier
  - More detail is provided on how this could be implemented
  - This would require building rustc with the beta standard library
    - At the time this was seen as unlikely, but has since been implemented
      ([rust#119899])
    - This implication was rejected as it was argued that `-Zbuild-std` would
      eventually allow a different source to be used
  - `-Zbuild-std` *might* be useful for bootstrap but using it in bootstrap is
    unlikely to help advance the stabilisation of build-std
    - There was detailed discussion about the degree to which `x.py` is just a
      wrapper around Cargo and whether the sysroot is necessary
  - Rust for Linux developers add that it is important for their use case that
    there is a simple way to build the standard library without requiring Cargo
    - Ideally with a mechanism for doing this using a stable toolchain
- **[wg-cargo-std-aware#77]: Cannot test bug fixes of dependencies of
  `libcore`/`libstd` on targets where `-Zbuild-std` is required**, [cr1901], Nov
  2021
  - User wants to change the `compiler_builtins` version to test upcoming
    bugfixes
  - Closed as duplicate of [wg-cargo-std-aware#7]

### Support for build-std in Cargo's subcommands
[support-for-build-std-in-cargo-subcommands]: #support-for-build-std-in-cargos-subcommands

These issues discuss the complexity involved in supporting build-std in specific
subcommands of Cargo:

- **[wg-cargo-std-aware#20]: Support `cargo metadata`**, [ehuss], Sep 2019
  - `cargo metadata` outputs machine-readable JSON containing an array of all of
    the packages in the workspace
    - The format has some stability guarantees
  - build-std would need to be supported in `cargo metadata` but the issue does
    not describe how that should work
  - There was no discussion on the issue
- **[wg-cargo-std-aware#21]: Support `cargo clean`**, [ehuss], Sep 2019
  - `cargo clean -p std` doesn't work but should
  - There was no discussion on the issue
- **[wg-cargo-std-aware#22]: Support `cargo fetch`**, [ehuss], Sep 2019
  - Implemented by [jyn514] in [cargo#10129]
    - Fetches crates.io dependencies of the standard library and should continue
      to do this if the dependencies are not vendored
- **[wg-cargo-std-aware#23]: Support `cargo vendor`**, [ehuss], Sep 2019
  - `cargo vendor` doesn't understand build-std
  - Vendoring the standard library and its dependencies would lock the user to a
    specific toolchain version, which would not be desirable
  - If `rust-src` contained a vendored standard library then `cargo vendor`
    would not need changed
    - Makes source replacement or `path` overrides challenging to implement
    - At the time of writing, vendoring the standard library was difficult as
      `cargo vendor` does not support vendoring a subset of the packages in a
      workspace, and the standard library did not have its own workspace at the
      time
  - Implemented by [Gankra] in [cargo#8834] and [rust#78790] by adding `patch`
    entries to all members to use the vendored crates
    - Later reverted in [cargo#8968] and [rust#79838] due to regressions
      - [cargo#8962]: Cargo was always updating the registry index with
        `-Zbuild-std`
      - [cargo#8963]: Cargo was producing unused patch warnings with
        `-Zbuild-std-features=compiler-builtins-mem`
      - [cargo#8945]: Custom targets were broken as Cargo didn't expect
        `rustc-std-workspace` in the vendored dependencies
    - Using patches to emulate vendoring (by changing `path`) is not correct and
      proper vendoring should be used
  - [rust#128534] moved the standard library to its own workspace which should
    make vendoring possible
    - There were some suggestions that using a vendored standard library should
      be optional as it prevents patching the standard library
- **[wg-cargo-std-aware#24]: Support `cargo pkgid`**, [ehuss], Sep 2019
  - `cargo pkgid` doesn't support build-std but should
  - There was no discussion on the issue
- **[wg-cargo-std-aware#26]: Possibly support `-p` for standard library
  dependencies**, [ehuss], Sep 2019
  - `-p` normally works for any dependency
  - There was discussion of how `-p` could work if `-Zbuild-std` were the final
    interface for build-std
  - There is a risk of leaking the dependencies of the standard library
    depending on how this support is implemented
- **[wg-cargo-std-aware#45]: Support --build-plan**, [ehuss], Sep 2019
  - `--build-plan` does not include anything from build-std
  - [cargo#7614] is a long and still-actively-discussed thread discussing the
    future of `--build-plan` with mixed opinions
  - `--build-plan` is still unstable so support for build-std can be a blocker
    for it if build-std lands first or vice-versa
- **[wg-cargo-std-aware#83]: Allow std to be specified as package spec**,
  [dullbananas], Apr 2023
  - User wants to use `-p` with build-std, closed as duplicate of
    [wg-cargo-std-aware#26]

### Dependencies of the standard library
[appendix-dependencies-of-the-standard-library]: #dependencies-of-the-standard-library

These issues discuss the challenges involved in building some of the
dependencies of the standard library:

- **[wg-cargo-std-aware#15]: Deps compiler_builtin**, [ehuss], Jul 2019
  - There's a `c` feature of this crate that uses optimised C versions for some
    of the intrinsics
    - This is used by default
  - The `mem` feature of this crate provides some demangled symbols used in
    `no_std` builds of `alloc` when there is not a `libc` to provide those
    symbols
  - Built using a large number of codegen units to force each function to into
    its own unit
    - [rust#135395] attempted to fix compiler-builtins' codegen unit
      partitioning in the compiler
      - It forced the compiler to use a large number of codegen units for the
        compiler-builtins crate
      - There is a tension between the embedded use-case for compiler-builtins'
        intrinsics and what Rust for Linux wants
        - RfL depends on compiler-builtins being compiled into a single object
          file
  - It was mentioned that almost all intrinsics now have a Rust implementation
    and that could be made the default for build-std eventually
    - Some concerns that this would result in divergence between locally-built
      and distributed standard libraries
- **[wg-cargo-std-aware#16]: Deps: backtrace**, [ehuss], Jul 2019
  - `libbacktrace` previously required a C compiler but has since been replaced
    by `gimli` in the standard library and so this is no longer an issue
    ([rust#46439])
- **[wg-cargo-std-aware#17]: Deps: sanitizers**, [ehuss], Jul 2019
  - At the time of writing, sanitizers are a dependency of the standard library
    and require LLVM components (located using `LLVM_CONFIG`)
  - It is suggested that the implementation of sanitizers could be revisited so
    that they are distributed as plain libraries alongside rustc and linking
    against them rather than rebuilding them as part of build-std
    - There would be no dependency on LLVM sources outside of building rustc.
      [rust#31605] could be resurrected to do this.
- **[wg-cargo-std-aware#18]: Deps: proc-macro**, [ehuss], Jul 2019
  - Procedural macros are already built with the host sysroot so wouldn't use a
    customised standard library
- **[wg-cargo-std-aware#65]: Does not work with vendoring**, [raphaelcohn], Feb
  2021
  - Closed as duplicate of [wg-cargo-std-aware#23]

### Design considerations
[design-considerations]: #design-considerations

These issues document open design questions for build-std:

- **[wg-cargo-std-aware#5]: Cargo standard library dependencies**, [ehuss], Jul
  2019
  - How will a dependency on the standard library be expressed in `Cargo.toml`?
  - There are various requirements laid about by the issue:
    - Users should have to opt-in to build the standard library
    - It should be able to support alternative standard library implementations
    - Cargo needs to be able to pass `--extern name` to specify explicit
      dependencies to add to the extern prelude, even for pre-build artifacts
  - References [rfcs#1133], [rfcs#2663], [cargo#5002], [cargo#5003], [rust#57288]
    - [rust#57288] tracks the effort to stop requiring `extern crate`. As of
      October 2023:
      - `--extern proc_macro` passed by Cargo to avoid needing
        `extern crate proc_macro`
      - `extern crate test` is still required, but `test` is unstable
      - `extern crate std`/`extern crate alloc` still required for `no_std` -
        Cargo has no way of knowing that a no_std crate uses std and that
        `--extern crate std` ought to be used
      - `extern crate core` as above for `no_core`
  - How does Cargo handle multiple crates in the graph declaring dependencies
    on the standard library?
    - Unify them, there is no concept of a "version" for the standard library
      dependencies
      - e.g. a union of all features
  - How to balance implicit vs explicit dependencies on the standard library?
    - Existing stable crates depend on the standard library implicitly
      - Tempting to declare every crate does unless opting out
    - Existing stable `no_std` crates do not have an explicit opt-out but are
      compatible with targets that work without a standard library at all
  - How to express a dependency on a pre-built artifact vs building from source?
    - Users could have no direct control, rebuilds are triggered based on other
      factors (e.g. profile settings, target, feature flags, etc) if a pre-built
      artifact does not exist for that configuration
    - Any differences between a locally-built artifact and pre-built artifact is
      a bug
  - It was suggested that the *Pre-Pre-RFC: making `std`-dependent Cargo
    features features a first-class concept* proposal be adopted (see below)
- **[wg-cargo-std-aware#6]: Target specification?**, [ehuss], Jul 2019
  - build-std makes target-spec-json more usable on stable and project teams
    will need to decide if they are comfortable with the current format or if it
    needs changed
  - Custom targets are more likely to require nightly as not having a standard
    library, they will need nightly to use build-std
  - If is opined that if build-std allows the standard library to be built for
    custom targets then that should require an explicit decision to stabilise
    the format
    - e.g. if `cargo +stable build --target=foo.json` works then the format is
      de-facto stable
  - The standard library's `build.rs` checks for whole target string which
    doesn't support custom targets very well
  - There are a handful of general requests of the target-spec-json format:
    - Switch to TOML
    - Clean up LLVM-specific details
    - Allow inherited specs
    - Use them in rustc rather than defining them in code
- **[wg-cargo-std-aware#8]: Standard library portability**, [ehuss], Jul 2019
  - Discusses needs for making the standard library more portable and configurable
    - i.e. not strictly build-std related but build-std benefits from a more
      portable standard library
  - References a bunch of previous/related efforts:
    - [rfcs#1502]
    - [internals.r-l.o: Fleshing out libstd scenarios]
    - [internals.r-l.o: Refactoring libstd for ultimate portability]
    - [rfcs#1868]
    - [A vision for portability in Rust]
    - [portability-wg]
    - [embedded-wg]
  - Somewhat out-of-scope for wg-cargo-std-aware
    - The mechanism for increasing portability may need to be exposed via
      `Cargo.toml` (e.g. features)
    - It is an impediment of making it easier to build std for unsupported targets
- **[wg-cargo-std-aware#11]: Downloading source**, [ehuss], Jul 2019
  - rustup can download `rust-src` which doesn't include the dependencies of the
    standard library, but does include the `Cargo.lock`
  - Preference that rustup not be required
    - Acquiring dependencies may be challenging
  - build-std needs to be transparent to be a first-class feature
    - Cargo has support for downloading various things but needs to know where
      the source is
    - rustup knows toolchain and commit but Cargo doesn't
    - Could have a reasonable default probing location (e.g. for distros) and
      could query rustup
  - [cargo#2768] allowed the user to specify a path in their configuration
  - Could publish the source for each standard library version to crates.io
- **[wg-cargo-std-aware#12]: `Cargo.lock` and dependency resolution**, [ehuss], Jul 2019
  - Very likely that the standard library will need to be built with the same
    dependency versions as distributed version
    - May end up with different versions of dependencies as the user's crate
  - Locking dependencies may be good to guarantee determinism
  - Closed assuming that this is a settled question
    - Current implementation uses a seperate resolve to keep dependency versions
      separate from user's dependency versions
- **[wg-cargo-std-aware#13]: Default feature specification**, [ehuss], Jul 2019
  - How to determine the default set of cfg values for the standard library?
    - Currently set by bootstrap
    - Also sets environment variables used by build scripts (e.g. `LLVM_CONFIG`)
  - Various possibilities
    - Hardcode in Cargo, not idea long-term
    - Some configuration in source distribution
    - Make user explicitly declare them
  - Similar to [wg-cargo-std-aware#4]
- **[wg-cargo-std-aware#14]: Default flags and environment variables**,
  [ehuss], Jul 2019
  - There are additional rustc flags passed to standard library crates which
    could be duplicated initially but not long-term
  - There are suggestions that a config file may be necessary as in
    [wg-cargo-std-aware#13]
  - There is overlap with [wg-cargo-std-aware#28]
  - There is a summary of the differences in flags between bootstrap and
    build-std in
    [wg-cargo-std-aware#14 (comment)][wg-cargo-std-aware#14-review]
- **[Pre-Pre-RFC: making `std`-dependent Cargo features features a first-class
  concept][wg-cargo-std-aware#5-internals]**, [bascule], Aug 2019
  - API guidelines say that Cargo features should be strictly additive and that
    gating std support should be behind a `std` feature
    - `no_std` users end up always using `default-features = false` and opting
      into everything except `std`
    - Unintentially enabled `std` features can prevent linking, often deep in
      dependency hierarchies
  - Proposes top-level `std-features = true` that automatically enables or
    disables the `std` features of dependencies
- **[wg-cargo-std-aware#29]: Support different panic strategies**, [ehuss], Sep 2019
  - Current implementation hardcoded for `unwind`
    - How does Cargo know what to use? Inspecting the profile?
    - Always build both `unwind`/`abort`?
    - `-Cpanic=abort` needs to be passed for `abort` crate
    - Some targets default to `abort`
    - `abort` crates currently rebuild w/ `unwind` when built with `libtest`
  - Cargo should work transparently - user sets `panic` in their profile and Cargo
    respects that and handles everything else
    - As everything is compiled by Cargo, can lift restrictions like `-Cpanic=abort`
      being necessary for `panic_abort` crate
    - Cargo should be able to take a more "pure" stance relative to `libtest`
      - [rust#64158] later merged supporting `panic=abort` with `libtest`
        - It is not yet stable ([rust#67650])
    - Ideally only compile one panic strategy crate
    - Target-specfic settings are tricky
      - Almost all cases of building the standard library from source are `panic=abort`
        targets w/ no unwinding support
  - Is profile sufficient to build `panic_abort`/`panic_unwind`?
    - Some crates don't want any panic strategy - when not using the standard library,
      not defining a panic strategy may make sense
    - `std` has `#![needs_panic_runtime]`
  - build-std may require `-Zpanic-abort-tests`
- **[wg-cargo-std-aware#30]: Figure out how to handle target-specific
  settings**, [ehuss], Sep 2019
  - Some targets have special logic and it isn't clear how this can be handled
    by Cargo
    - Often target-specific and toolchain-related
  - Can potentially whitelist or blacklist targets
- **[wg-cargo-std-aware#38]: Consider doing a forced lock of the standard
  library**, [ehuss], Sep 2019
  - Ensure that the in-memory `Cargo.lock` is not modified
  - Hard to guarantee in practice
    - Could add a test to assert this
      - [cargo#13916] attempted to add a check to verify the virtual workspace
        for the standard library post-resolve is a strict subset of it
        pre-resolve
  - [rust#128534]/[cargo#14358] meant that the standard library has its own
    workspace and build-std does not need to generate one
    - On-disk lockfile is now used so do not need to worry about it changing
  - Still need to implement `--locked` behaviour
    - Doing so would break `[patch]` with build-std
- **[wg-cargo-std-aware#39]: Figure out interaction with public/private dependencies**, [ehuss], Sep 2019
  - Public/private status of the standard library is hardcoded to public in MVP - should
    it be?
  - If defined in `Cargo.toml`, it will likely inherit visibility from that, which defaults
    to private.
- **[wg-cargo-std-aware#43]: What will the UX be for enabling "build libstd"
  logic?**, [alexcrichton], Sep 2019
  - How will build-std eventually be enabled?
  - At time of writing, `-Zbuild-std=$crates`, but what should the eventual
    syntax be?
    - e.g. `build.std` option in `Cargo.toml` or `.config/cargo`
  - Opting into build-std as an unstable feature should be a `-Z` flag or entry
    in `cargo-features`
  - Eventually not a flag at all, and Cargo should try to compile the standard
    library automatically if a compiled standard library is not available
    - Concern that the pre-compiled standard library may not match by default
      and there would be too many false-positive rebuilds
    - For missing targets, need to distinguish between forgetting to run
      `rustup target add` and needing to compile the standard library
    - `[profile]` could not affect the standard library by default
      - There are cases where implicit rebuilding could be desirable
        - e.g. ABI-modifying flags
  - Passing a flag on every invocation is not a good user experience
  - [cargo#10308] is a prototype with explicit standard library dependencies
    - Idea posted on internals, see *Build-std and the standard library* below
  - If enabled manually, [cargo#8733] expressed the need to enable build-std
    only for certain targets, in particular when cross-compiling to no-std
    targets
- **[wg-cargo-std-aware#46]: How to handle special object files?**, [ehuss], Sep 2019
  - How to handle pre-built object files needed to link on a target?
    - e.g. `crt1.o`, `crti.o`, `crtn.o`, etc for musl or `dllcrt2.o`,
      `crt2.o`, etc for Windows GNU
  - Cargo is target agnostic and does not want to hardcode logic for particular
    targets
    - The target definitions define which files they expect to find and link
      against
  - [rust#68887] adds a `self-contained` directory which could help
- **[wg-cargo-std-aware#47]: how to handle pre-built linkers?**, [ehuss], Sep
  2019
  - Some pre-built targets ship with a copy of rust-lld or gcc to assist with
    linking
  - `rust-lld` could be shipped as a rustup component or the user could be
    forced to install these components
    - Since this issue was filed, `rust-lld` is now always shipped
  - rustc finds `rust-lld` via the sysroot and if the sysroot is not provided
    then another mechanism will need to be used to find `rust-lld`
- **[wg-cargo-std-aware#50]: Impact on build scripts that invoke rustc**, [jdm], Oct 2019
  - Need to make sure that build scripts that invoke rustc are able to do this with the correct
    standard library
  - Closed as t-cargo do not want to encourage or support build probes
- **[wg-cargo-std-aware#51]: Plan for removal of `rustc-dep-of-std`**, [ehuss], Nov 2019
  - `rustc-dep-of-std` is a feature that packages used as dependencies of `std` have that,
    when enabled, use to declare explicit dependencies of `core`/`compiler_builtins`/etc
    - May not be necessary once these packages are always declaring dependencies on
      `core`/etc
    - Cargo would still need to work out what to do when seeing explicit dependencies
  - These dependencies could always be built with build-std but that would involve
    using build-std in bootstrap which may not be desirable
  - Alternatively, have some mechanism for telling Cargo that the explicit dependencies
    in this instance aren't from the sysroot but an rlib from bootstrap
    - Could be automatic or variation on patch syntax
- **[wg-cargo-std-aware#57]: Support building workspace with separate build-std
  parameters**, [jschwe], Jun 2020
  - User wants to have one package depend on `core` and `alloc` and another
    package depend on `core`, `alloc` and `std` but this isn't possible with the
    current experimental flag
  - Closed as duplicate of [wg-cargo-std-aware#5]
- **[wg-cargo-std-aware#68]: Support Profile Guided Optimisation (PGO)**, [errantmind], Mar 2021
  - `profiler_builtins` isn't compatible with `no_core`
    - i.e. `profiler_builtins` requires `core` so if building with profiling, need to
      build `core` first w/out the profiling instrumentation
  - [rust#79958] improves the error message for this
- **internals.r-l.o: [Build-std and the standard library][wg-cargo-std-aware#43-internals]**, [ehuss], Dec 2019
  - Proposes changing the standard library so all of its crates can build on all
    targets
    - It may not work and not all APIs will be available, but there will be no
      compilation errors
      - i.e. add runtime errors or cfgs
    - Makes things simpler because build-std can become a simple toggle
  - Building an empty standard library is preferred over not building at all as
    this avoids breaking `no_std` crates that build for tier three targets with
    no standard library support at all
  - Suggestion that the standard library use Cargo features more so that most
    functionality is gated by default
    - Fixes this issue as most of the standard library would be absent by
      default
  - Some prefer that the standard library be listed in the `Cargo.toml`
    eventually but that this would work initially
    - Explicit standard library dependencies are being considered
      - It could be a inconvenience to need to specify the standard library
        dependencies in most cases
        - Most packages don't need it
      - Need default for packages that do not specify a standard library
        dependency
  - References to the "portability lint"
    - Lint that triggers when calling an available platform-specific code so
      that users can avoid making code less portable
    - Lint was later deemed infeasible
  - Described as necessary step towards long-standing goal to move from a
    multiple-crates model of the standard library to a cfg-flag model
  - Cargo could learn what targets a package supports
    - e.g. libstd could declare this
    - Doesn't work for custom targets (i.e. target-spec-json)
    - Could accidentally allow the standard library to be used in `no_std`
      projects
    - Adds noise to `Cargo.toml`
- **[wg-cargo-std-aware#85]: Figure out rust-lang/rust testing strategy**, [ehuss], Mar 2023
  - How to test build-std when it is tightly coupled to the standard library source tree?
    - Could use rust-toolstate but wasn't very good
    - Could use git subtrees (or [JOSH])
    - Could aim to reduce coupling between the standard library and Cargo
  - Which targets get tested and how?
- **[wg-cargo-std-aware#86]: Consider limiting supported targets in initial
  stabilization**, [ehuss], Mar 2023
  - Supporting build-std on all targets is a large effort so consider
    supporting fewer targets initially
  - There could be a concept of stability for targets w/ build-std
    - i.e. this target works with build-std on nightly only
  - Generally supportive reception but every contributor wanted their target to be
    one of the supported ones
- **[wg-cargo-std-aware#88]: `cargo doc -Zbuild-std` doesn't generate. links to the standard library**, [jyn514], Jun 2023
  - Cargo doesn't treat the standard library as coming from crates.io and the standard
    library doesn't have `html_root_url` set, so local documentation doesn't get links to
    the standard library
    - Bootstrap passes `-Zcrate-attr="doc(html_root_url=..)"`
- **[cargo#12375]: artifact-dependencies doesn't compose well with build-std**, [tamird], Jul 2023
  - The current unstable implementation of build-std is required when building
    for tier three targets but it is impossible to enable it for an artifact
    dependency.
- **[wg-cargo-std-aware#90]: restricted_std applicability to custom JSON targets**, [Mark-Simulacrum], Feb 2024
  - The standard library's `build.rs` changed to use `TARGET_OS` rather than an complete
    target name so JSON targets don't end up enabling `restricted_std`
    - Need to work out what guarantees are desired
  - Should std-supported be a property of the target, not `build.rs`?
  - Idea behind the current approach:
    - Slightly modified target specs should continue to work and permit usage of the
      standard library
    - Different target specs that are very different from existing targets ought not
      permit usage of the standard library
  - [rust#71009] needs to be resolved first
    - Aims to de-stabilise target specifications, making it prohibited to pass one to
      the compiler
    - cc [wg-cargo-std-aware#6]
  - If target-spec-json is unstable then their behaviour with build-std is also unstable
- **[wg-cargo-std-aware#92]: Consider flag to mark unstable targets**, [madsmtm], Feb 2024
  - Tier three targets are required to use nightly because of build-std
    - Stable build-std effectively stabilises these targets, so users would stop using
      nightly and get fixes less frequently
  - Is a notion of unstable or incomplete targets desirable?
- **[wg-cargo-std-aware#95]: Use `build-std=core` on a custom target
  automatically**, [nazar-pc], Mar 2024
  - Closed as duplicate of [wg-cargo-std-aware#43]

### Implementation
[implementation]: #implementation

These issues include bug reports for the current unstable implementation of
build-std, as well as unresolved questions for the implementation, both from
[wg-cargo-std-aware]. In addition, this section will list the pull requests in
[rust-lang/cargo] and [rust-lang/rust] that directly contributed to the
implementation.

- **[cargo#7216]: Basic standard library support.**, [ehuss], Aug 2019
  - Initial implementation of `-Zbuild-std`
  - Constructed a synthetic workspace for the standard library and used
    `--extern` to provide the new dependencies to rustc.
  - Required `--target` to be passed so that the host sysroot was used for build
    scripts and procedural macros.
- **[cargo#7336]: Add `alloc` and `proc_macro` to libstd crates**, [alexcrichton], Sep 2019
  - Ensures `alloc` and `proc_macro` are also built when `std` is
- **[wg-cargo-std-aware#25]: Remove requirement for `--target`**, [ehuss], Sep 2019
  - `--target` must be specified to require Cargo run in cross-compilation mode (as
    the `proc_macro` crate needs the host sysroot)
  - Fixed in [cargo#14317]
- **[wg-cargo-std-aware#27]: Possibly publish a synthetic `Cargo.toml`**, [ehuss], Sep 2019
  - build-std initially had to create a false `Cargo.toml` for the standard library as it
    was part of a workspace with the rest of rust-lang/rust
  - Fixed by [cargo#14358] (after [rust#128534])
- **[wg-cargo-std-aware#28]: Fixup some crate flags**, [ehuss], Sep 2019
  - Some crates require special compiler flags which are scattered throughout bootstrap
    - e.g. `compiler_builtins` uses `debug-assertions=no`, `codegen-units=1`, `panic=abort`
      - cc [wg-cargo-std-aware#15]
  - There will always be some implicit contract between the standard library and build-std
    - Move these to `Cargo.toml` wherever possible
      - Progress made in [rust#64316]
      - Bootstrap has been improved so there is less of this - target-specific and
        sanitizer parts remain
- **[wg-cargo-std-aware#31]: Possibly add a way to disable the sysroot on
  `rustc`**, [ehuss], Sep 2019
  - Implementation uses `--extern` to tell rustc where to find the standard library but it
    will look in the sysroot for any crate that was not provided with an `--extern`
    argument
    - This can end up loading sysroot versions of the standard library and with
      duplicate language item errors
  - Was closed when build-std used `--sysroot` in [cargo#7421] and then re-opened when
    it changed back to `--extern` in [cargo#7699]
- **[wg-cargo-std-aware#33]: Consider better testing strategy**, [ehuss], Sep 2019
  - build-std's testing in Cargo after the initial implementation wasn't very good and has
    since been improved
- **[wg-cargo-std-aware#34]: Consider mixing `__CARGO_DEFAULT_LIB_METADATA` into the
  hash**, [ehuss], Sep 2019
  - Environment variable is used for embedding the release channel into the metadata
    hash but does not appear to be necessary for build-std
- **[wg-cargo-std-aware#35]: Consider not building std as a dylib**, [ehuss], Sep 2019
  - The standard library's manifest builds it as both a rlib and a dylib
    - build-std only needs the rlib
    - Don't want to produce the dylib and have anyone depend on it always being built
  - Fixed by [cargo#7353]
- **[wg-cargo-std-aware#37]: Consider setting `require_optional_deps` to `false` for
  standard library**, [ehuss], Sep 2019
  - Workspace created for the standard library sets `require_optional_deps` to `true`
    but probably doesn't need to
  - Fixed in [cargo#7337]
- **[cargo#7337]: Don't resolve std's optional dependencies**, [alexcrichton], Sep 2019
  - Unrequested optional dependencies are typically the `dev-dependencies` of
    `std` and so don't need to be built
  - Fixes [wg-cargo-std-aware#37]
- **[cargo#7350]: Improve test suite for -Zbuild-std**, [alexcrichton], Sep 2019
  - Many improvements to build-std testing, primarily the introduction of a
    "mock-std" workspace which mimics the standard library's structure and
    allows tests to run much quicker
  - Fixes [wg-cargo-std-aware#33]
- **[rust#64316]: Delete most of `src/bootstrap/bin/rustc.rs`**, [alexcrichton], Sep 2019
  - Moved most of the standard library's configuration from the rustc shim in
    bootstrap to the standard library using Cargo features, so it could be
    leveraged by `-Zbuild-std`
- **[wg-cargo-std-aware#40]: Using `--extern` is apparently not equivalent to
  `--sysroot`**, [alexcrichton], Sep 2019
  - Using `--extern alloc=alloc.rlib` meant that writing `extern crate alloc` was
    not required, which is different than putting `alloc.rlib` in the sysroot where
    `extern crate alloc` would still be required
    - A consequence of this is that a locally compiled standard library would not
      require `extern crate alloc` while the pre-compiled standard library would
  - build-std switched to using the sysroot in [cargo#7421] and then switched back to
    using `--extern` in [cargo#7699] (once `--extern noprelude:alloc=alloc.rlib` was
    added)
- **[cargo#7421]: Change build-std to use --sysroot**, [ehuss], Sep 2019
  - The initial implementation used `--extern` to provide rustc the newly-built
    standard library artifacts to later rustc invocations. This did not have
    identical behaviour to the existing pre-built artefacts in the sysroot
    ([wg-cargo-std-aware#40])
  - Negated the need to prevent rustc from using the sysroot
    ([wg-cargo-std-aware#31])
  - Instead, this PR constructed a sysroot in Cargo's `target` directory and passed
    that to rustc to replace the default sysroot
  - It was found that this sysroot approach could still allow users to depend on
    sysroot crates without declaring a dependency on
    it([wg-cargo-std-aware#49])
- **[wg-cargo-std-aware#41]: Documentation on how to use the new cargo-std support in
  place of cargo xbuild**, [alex], Sep 2019
  - Fixed by adding documentation on build-std to Cargo's unstable feature documentation
- **[wg-cargo-std-aware#44]: Disable incremental for std crates**, [ehuss], Sep 2019
  - Incremental is not necessary for build-std
  - Fixed in [cargo#8177]
- **[wg-cargo-std-aware#48]: Investigate custom libdir setting**, [ehuss], Sep 2019
  - Cargo expects a specific sysroot layout that can be changed with `bootstrap.toml`
  - Intended to switch to `--print=target-libdir` from [rust#69608]
    - Later made irrelevant by [cargo#7699]
- **[wg-cargo-std-aware#49]: Usage of `--sysroot` may still be racy and/or allow false
  dependencies**, [alexcrichton], Oct 2019
  - Later made irrelevant by [cargo#7699] where `--extern` is used again
- **[cargo#7699]: Switch build-std to use --extern**, [ehuss], Dec 2019
  - A revert of [cargo#7421], but uses new `--extern` options `priv` and
    `noprelude` from [rust#67074]
  - Adding standard library crates to the extern prelude was the cause of
    [wg-cargo-std-aware#40]
  - Re-opened [wg-cargo-std-aware#31] but fixes [wg-cargo-std-aware#49]
- **[wg-cargo-std-aware#53]: `compiler_builtins` seems to be missing
  symbols**, [tomaak], Jan 2020
  - `compiler_builtins` provides symbols through the `mem` feature
    - It is not enabled by default as the standard library typically gets
      these symbols through the platform `libc`
      - `libc`'s implementation is typically better optimized
    - Some `no_std` targets need compiler_builtins' implementations
  - A concept of "target-specific features" could be beneficial
    - Supporting custom targets makes this tricky
  - `compiler-builtins-mem` feature is forwarded through standard library
    crates to `compiler_builtins`
  - [compiler-builtins#411] added weak linkage to the mem functions in compiler-builtins
- **[cargo#7931]: build-std: remove sysroot probe**, [ehuss], Feb 2020
  - An optimisation to remove an unncecessary `rustc --print` invocation
- **[wg-cargo-std-aware#54]: Adjust libstd to make non-Rust dependencies
  optional**, [nagisa], Apr 2020
  - Cross-compiling the standard library's `C` dependencies is difficult
  - The standard library now uses `gimli` not backtrace which makes this easier
  - `libunwind` can be linked with `-Clink-self-contained`
  - linux-musl and fortanix-sgx both may still require a C compiler
    - Unclear if this can be avoided
- **[wg-cargo-std-aware#55]: Persistent unused attribute warnings when recompiling
  libcore after linker flags are changed**, [cr1901], Apr 2020
  - Issue with incremental compilation, fixed by [cargo#8177] (cc [wg-cargo-std-aware#44])
- **[cargo#8177]: build-std: Don't treat std like a "local" package**, [ehuss], Apr 2020
  - Adds the concept of a "local" package (not controlled by the user)
  - `build-std` no longer uses incremental or dep-info fingerprint tracking and
    will not show warnings in standard library crates
  - Closed [wg-cargo-std-aware#44] and [wg-cargo-std-aware#55]
- **[wg-cargo-std-aware#62]: Linker can't find `core::panicking::panic` when lto is turned
  on**, [hnj2], Nov 2020
  - `panic` can't be found with LTO turned on
  - compiler-builtins wasn't being built with `overflow-checks=false` and
    `debug-assertions=false`
- **[cargo#8490]: Add a `-Zbuild-std-features` flag**, [alexcrichton], Jul 2020
  - Allows users to enable Cargo features from the standard library
- **[rust#77086]: Include libunwind in the rust-src component**, [ehuss], Sep 2020
  - Includes `src/llvm-project/libunwind` in `rust-src` which is needed by some
    targets, such as musl targets, to build the unwind crate
- **[cargo#8834]: Patch in vendored dependencies in `rust-src`**, [Gankra], November 2020
  - A step towards supporting `cargo vendor` ([wg-cargo-std-aware#23])
  - Allow for the `rust-src` component to include vendored dependencies of the standard
    library
  - Reverted in [cargo#8968] due to:
    - [cargo#8962]: `-Zbuild-std` always updates the registry index
    - [cargo#8963]: unused patch warnings when using
      `-Zbuild-std-features=compiler-builtins-mem`
    - [cargo#8945]: `-Zbuild-std` with custom targets was broken
- **[rust#78790]: Vendor libtest's dependencies in the rust-src component**, [Gankra], Nov 2020
  - [rust-lang/rust] half of [cargo#8834]
  - Reverted in [rust#80082] as it caused `x.py dist` to always require
    network access ([rust#79218])
- **[wg-cargo-std-aware#63]: Support code-coverage**, [catenacyber], Dec 2020
  - Finding duplicate language item when building with `-Zinstrument-coverage` and build-std
  - Works with `-Zno-profiler-runtime`
    - Presumably profiler runtime is being loaded from the sysroot and that is loading other
      sysroot crates and conflicting with the locally built crates
- **[wg-cargo-std-aware#64]: -Z build-std with unified workspace**, [Ericson2314], Jan 2021
  - In the current build-std implementation, the standard library is resolved
    separately then combined with the user's crate graph
    - It is argued that this is undesirable as it makes the standard library special
  - Better to keep them separate as the standard library wants to have fixed dependency
    versions matching the distributed version
- **[wg-cargo-std-aware#66]: Cross-compilation (of libc) on MacOS fails**, [raphaelcohn], Feb 2021
  - Unclear exactly what the root cause of this issue is
- **[wg-cargo-std-aware#67]: Update README.md**, [ghost], Mar 2021
  - Merged into [wg-cargo-std-aware]
- **[wg-cargo-std-aware#69]: "use of unstable library feature 'restricted_std'" can't be
  fixed for deps**, [Manishearth], Jun 2021
  - User wants to use crates which use the standard library but where `restricted_std` applies
    and that can't be fixed for the dependencies other than by patching
- **[wg-cargo-std-aware#70]: error: could not find native static library `c`, perhaps a `-L`
  flag is missing?**, [mkb2091], Jun 2021
  - Duplicate of [wg-cargo-std-aware#66]
- **[wg-cargo-std-aware#72]: Figure out how to deal with `cargo test` with a
  `no-std` target**, [phip1611], Jul 2021
  - `cargo test` doesn't work with `no_std` targets as `restricted_std` triggers
    on `libtest`
  - There were no comments on this issue
- **[cargo#10129]: Add support for `-Zbuild-std` to `cargo fetch`**, [jyn514], Nov 2021
  - Enables `cargo fetch -Zbuild-std` to fetch standard library crates
- **[wg-cargo-std-aware#76]: Unable to build executable for musl target**, [HenryJk], Nov 2021
  - It is unclear what fixed this issue, potentially a libc version bump
    or linking `self-contained`
- **[cargo#10308]: Move build-std to Cargo.toml**, [fee1-dead], Jan 2022
  - Attempts to fix issues with build-std and per-package-target ([cargo#9451])
  - build-std's user interface is a large and open question that this patch
    didn't have all the answers for so this was later closed
    ([wg-cargo-std-aware#43])
- **[cargo#10330]: Support per pkg target for -Zbuild-std**, [fee1-dead], Jan 2022
  - Another attempt to fix build-std and per-package-target ([cargo#9451])
  - Attempted to remove `--target` restriction but was told that this wasn't possible
    - It probably was possible, thanks to various refactorings to Cargo between
      2019 and 2022, as [cargo#14317] faced no difficulties in removing the
      restriction
- **[wg-cargo-std-aware#81]: -lunwind despite build-std=\["panic_abort", "std"\] on
  powerpc-unknown-linux-musl**, [george-hopkins], Oct 2022
  - It is unclear what this issue is, there was very little detail and nobody commented
- **[rust#108924]: panic_immediate_abort requires abort as a panic strategy**, [tmiasko], Mar 2023
  - Adds a compile error when the `panic_immediate_abort` feature isn't used with `-Cpanic=abort`
  - This could be triggered by build-std, as per [rust#107016]
- **[cargo#12088]: hack around `libsysroot` instead of `libtest`**, [weihanglo], May 2023
  - Cargo previously resolved features from the `test` crate, now it does so from
    the `sysroot` crate, which is the canonical "head" of the standard library
  - This approach to resolving features of standard library crates is still
    considered a hack
- **[wg-cargo-std-aware#87]: The restricted_std error message is confusing**, [ehuss], May 2023
  - If you build a `no_std` target and forget to include `#![no_std]` then the standard library
    is loaded and there's an "unstable library feature" error, this is confusing
  - Improved error added in [rust#123360] and checking if a target supports the standard library
    in [cargo#14183]
- **[cargo#13065]: fix: reorder `--remap-path-prefix` flags for `-Zbuild-std`**, [weihanglo], Nov 2023
  - Changing the order these flags are passed improves the source path in
    diagnostics
- **[rust#120232]: Add support for JSON targets when using build-std**, [c272], Jan 2024
  - Updates the `restricted_std` filtering in `std`'s
    [`build.rs`][std-build.rs] to check the target os rather than the target
    triple (which is just set to the filename for JSON targets).
  - Custom targets that "look similar" to builtin targets do not need to use
    `restricted_std` ([wg-cargo-std-aware#90])
- **[cargo#13404]: Verify build-std crate graph against library lock file**, [c272], Feb 2024
  - Added a new Cargo test, as an alternative to doing a forced lock
    ([wg-cargo-std-aware#38]), to ensure that the resolved standard library unit
    graph is a subset of the distributed `Cargo.lock`
  - Closed in preference of a check at runtime
- **[rust#123360]: Document restricted_std**, [adamgemmell], Apr 2024
  - Improves the error message encountered when attemping to use `std` on a
    target where `restricted_std` is set
  - Proposed closing issues around `restricted_std` but a more comprehensive
    solution was desired
- **[cargo#13916]: Verify build-std resolve against original lockfile**, [adamgemmell], May 2024
  - Same as [cargo#13404] but during Cargo execution
  - Superseded by changes from [rust#128534]
- **[cargo#14183]: Check build target supports std when building with -Zbuild-std=std**, [harmou01], Jul 2024
  - Disallows building `std` when `metadata.std` field in the unstable
    `target-spec-json` is `false`
  - Aims to improve user experience compared to the `restricted_std` error
  - `target-spec-json`'s `metadata.std` was added to support generation of the
    compiler documentation, rather than as a source-of-truth for this
    information
- **[cargo#14317]: Remove requirement for --target when invoking Cargo with -Zbuild-std**, [harmou01], Jul 2024
  - Cargo now defaults to "cross-compile" mode and unifies units when in
    "host-only" mode, so host dependencies do not use the build-std standard
    library and this restriction can be removed
- **[rust#128534]: Move the standard library to a separate workspace**, [bjorn3], Aug 2024
  - `rust-src` now has its own lockfile, enabling simplifications in build-std
    implementation
- **[cargo#14358]: Remove hack on creating virtual std workspace**, [weihanglo], Aug 2024
  - Following [rust#128534], Cargo does not need to construct a synthetic
    workspace and can load the workspace from disk
  - Also enables `build-std` to use the configuration present in the workspace
    manifest
- **[cargo#14370]: fix: std Cargo.lock moved to `library` dir**, [weihanglo], Aug 2024
  - Fix for [cargo#14358] which use the new `library/Cargo.lock`
- **[wg-cargo-std-aware#91]: rust-lld: undefined symbol: memchr when compiling
  with nightly-aarch64-unknown-linux-musl**, [yogh333], Aug 2024
  - Undefined `memchr` symbol on `aarch64-unknown-linux-musl` when using `compiler-builtins-mem`
    feature
  - Suggested that the issue is a missing aarch64 musl libc
- **[cargo#14589]: Implement `--locked` for build-std**, [adamgemmell], Sep 2024
  - Alternative to [cargo#13916]
  - Reuses Cargo's `--locked` machinery now that the standard library has its
    own workspace and lockfile
  - Concerns raised about resolving with optional dependencies and breaking the
    future ability to patch the standard library workspace
  - Closed pending a more comprehensive plan from the build-std project goal
- **[cargo#14850]: always link to std when testing proc-macros**, [weihanglo], Nov 2024
  - A small fix for testing proc-macros with build-std when `libstd.so` stopped
    being shipped
- **[cargo#14899]: determine root crates by target spec `std:bool`**, [weihanglo], Dec 2024
  - This removes the hard error from [cargo#14183] and instead uses `std` as the
    default crate for build-std if `metadata.std` is true, and
    `core`/`compiler_builtins` otherwise
  - `std` can be built on some targets even if they don't officially support
    std, and rustdoc was relying on this behaviour
- **[cargo#14938]: make Resolve align to what to build**, [weihanglo], Dec 2024
  - Reverted part of [cargo#14899] which meant that `panic_unwind` would not
    build if the `panic-unwind` feature was not present
- **[cargo#14951]: Do not hash absolute sysroot path into stdlib crates metadata**, [Dirbaio], Dec 2024
  - Improves reproducibility of `build-std` builds by only hashing paths of
    standard library sources relative to the sysroot
- **[cargo#15065]: parse as comma-separated list**, [weihanglo], Jan 2025
  - Fixes a minor regression when providing multiple crates via
    `CARGO_UNSTABLE_BUILD_STD`
- **[rust#135395]: Enforce the compiler-builtins partitioning scheme**, [saethlin], Jan 2025 (closed)
  - Removes the profile override for `compiler_builtins`' codegen-units and
    implements it in the compiler instead
  - One of the use cases is build-std, which does not use the profile override
  - Closed due to build issues with Rust for Linux unrelated to build-std
- **[wg-cargo-std-aware#93]: Stack trace for duplicate lang item?**, [illuzen], Feb 2025
  - Missing `panic_abort` and so loading it from the sysroot and hitting a duplicate language
    item error
  - Downstream of [wg-cargo-std-aware#31]
- **[wg-cargo-std-aware#94]: `panic_immediate_abort` and `no_std`**, [nazar-pc], Mar 2025
  - Suggests that a panic handler crate shouldn't be necessary when
    `panic_immediate_abort` is enabled

### Bugs in the compiler or standard library
[bugs-in-the-compiler-or-standard-library]: #bugs-in-the-compiler-or-standard-library

These issues were bug reports for build-std that ultimately ended up being
issues resolved in the standard library or compiler:

- **[wg-cargo-std-aware#32]: Figure out why profile override causes linker
  errors**, [ehuss], Sep 2019
  - Ended up being a bug in symbol mangling ([rust#64319])
- **[wg-cargo-std-aware#52]: cannot produce proc-macro on musl host toolchain**,
  [12101111], Nov 2019
  - Issue was entirely unrelated to build-std
- **[wg-cargo-std-aware#56]: duplicate item in crate `core`**, [chaozju], Jun
  2020
  - User's dependency was using `std` (forgot to disable `std` feature)
- **[wg-cargo-std-aware#58]: It is not possible to use `-Zbuild-std` with a
  Development-Channel Rust**, [cr1901], Aug 2020
  - User's toolchain version did not match local checkout of [rust-lang/rust]
- **[wg-cargo-std-aware#59]: Can't build executables for musl**, [vi], Sep 2020
  - `libunwind`'s source was missing in `rust-src` for targets that need it
    - Fixed in [rust#77086]
- **[wg-cargo-std-aware#60]: Can't build std if I specify target json file: std
  does not see networking**, [vi], Sep 2020
  - The standard library's `build.rs` was matching on entire target names rather
    than just components like `target_os`
    - Fixed in [rust#120232]
- **[wg-cargo-std-aware#71]: "duplicate lang item in crate `core`" when
  building**, [TheBlueMatt], Jul 2021
  - Duplicate of [wg-cargo-std-aware#56]
- **[wg-cargo-std-aware#73]: Build on Windows fails to select a version of
  `libc` for package `test`**, [MauriceKayser], Oct 2021
  - Duplicate of [cargo#9976], ultimately unrelated to build-std
- **[wg-cargo-std-aware#74]: undefined reference errors on aarch64**,
  [SparrowLii], Nov 2021
  - `core`'s implementation briefly depended on libc after [rust#83655]
    - `compiler_builtins` had no implementation of the symbols that the
      `outline-atomics` feature was using from `libc`
    - `compiler_builtins` gained implementation in [compiler-builtins#532]
- **[wg-cargo-std-aware#75]: Code won't compile with panic="abort" option**,
  [HenryJk], Nov 2021
  - User was missing `panic_abort` crate in `-Zbuild-std=`
- **[wg-cargo-std-aware#78]: Rust compiler workspace patches are being ignored
  when compiling with `-Zbuild-std`**, [raoulstrackx], Nov 2021
  - Adding a `[patch]` for the standard library to the workspace in the
    [rust-lang/rust] did not work
  - Fixed when the standard library gained its own workspace ([rust#128534])
- **[wg-cargo-std-aware#79]: Building std with support for certain lang items**,
  [AZMCode], Jan 2022
  - User wanted to define their own language item for `Error` in a `no_std`
    project
  - Ultimately `Error` was moved to `core`
- **[wg-cargo-std-aware#80]: duplicate lang item if `#![feature(test)]` is
  enabled**, [skyzh], Mar 2022
  - `std` wasn't in the list of crates to `-Zbuild-std` and was being pulled from sysroot,
    so `core` was being loaded twice
- **[wg-cargo-std-aware#82]: Hidden symbol isn't defined**, [wcampbell0x2a], Jan
  2023
  - Duplicate of [rust#107016], fixed in [rust#108924]

### Cargo feature requests narrowly applied to build-std
[cargo-feature-requests-narrowly-applied-to-build-std]: #cargo-feature-requests-narrowly-applied-to-build-std

These issues were feature requests for build-std that could have been a more
general feature for Cargo that could then apply to build-std too:

- **[wg-cargo-std-aware#84]: Cache libstd artifacts between projects**,
  [jyn514], May 2023
  - Cargo rebuilds the standard library for each project used with `-Zbuild-std`
    (in the `target` directory) but it could be cached globally
  - It was the opinion of team members that this was hard to implement
  - It also could apply to any dependency of specific version and configuration
    and so this could be resolved by proposing the addition of a global caching
    mechanism for dependencies
- **[wg-cargo-std-aware#89]: Allow scoping of unstable features to specific
  targets**, [ketsuban], Oct 2023
  - User wants unstable `build-std` to only be enabled for one target and
    nightly not be required when building for other targets
  - This is a consequence of how Cargo's unstable features work, and would be
    fixed by a change to that mechanism, rather than anything specific to
    build-std

[JOSH]: https://josh-project.github.io/josh/intro.html
[cargo-xbuild]: https://github.com/rust-osdev/cargo-xbuild
[embedded-wg]: https://github.com/rust-embedded/wg
[panic-abort]: https://crates.io/crates/panic-abort
[panic-halt]: https://crates.io/crates/panic-halt
[panic-itm]: https://crates.io/crates/panic-itm
[panic-semihosting]: https://crates.io/crates/panic-semihosting
[portability-wg]: https://github.com/rust-lang-nursery/portability-wg
[rust-lang/cargo]: https://github.com/rust-lang/cargo
[rust-lang/crates.io-index]: https://github.com/rust-lang/crates.io-index
[rust-lang/rust]: https://github.com/rust-lang/rust
[wg-cargo-std-aware]: https://github.com/rust-lang/wg-cargo-std-aware
[xargo]: https://github.com/japaric/xargo

[A vision for portability in Rust]: http://aturon.github.io/tech/2018/02/06/portability-vision/
[Opaque dependencies]: https://hackmd.io/@epage/ByGfPtRell

[cargo#10129]: https://github.com/rust-lang/cargo/pull/10129
[cargo#10308]: https://github.com/rust-lang/cargo/pull/10308
[cargo#10330]: https://github.com/rust-lang/cargo/pull/10330
[cargo#10881]: https://github.com/rust-lang/cargo/issues/10881
[cargo#12088]: https://github.com/rust-lang/cargo/pull/12088
[cargo#12375]: https://github.com/rust-lang/cargo/pull/12375
[cargo#13065]: https://github.com/rust-lang/cargo/pull/13065
[cargo#13404]: https://github.com/rust-lang/cargo/pull/13404
[cargo#13916]: https://github.com/rust-lang/cargo/pull/13916
[cargo#14183]: https://github.com/rust-lang/cargo/pull/14183
[cargo#14317]: https://github.com/rust-lang/cargo/pull/14317
[cargo#14358]: https://github.com/rust-lang/cargo/pull/14358
[cargo#14370]: https://github.com/rust-lang/cargo/pull/14370
[cargo#14589]: https://github.com/rust-lang/cargo/pull/14589
[cargo#14850]: https://github.com/rust-lang/cargo/pull/14850
[cargo#14899]: https://github.com/rust-lang/cargo/pull/14899
[cargo#14938]: https://github.com/rust-lang/cargo/pull/14938
[cargo#14951]: https://github.com/rust-lang/cargo/pull/14951
[cargo#15065]: https://github.com/rust-lang/cargo/pull/15065
[cargo#2768]: https://github.com/rust-lang/cargo/pull/2768
[cargo#4959]: https://github.com/rust-lang/cargo/issues/4959
[cargo#5002]: https://github.com/rust-lang/cargo/issues/5002
[cargo#5003]: https://github.com/rust-lang/cargo/issues/5003
[cargo#7216]: https://github.com/rust-lang/cargo/pull/7216
[cargo#7336]: https://github.com/rust-lang/cargo/pull/7336
[cargo#7337]: https://github.com/rust-lang/cargo/pull/7337
[cargo#7350]: https://github.com/rust-lang/cargo/pull/7350
[cargo#7353]: https://github.com/rust-lang/cargo/pull/7353
[cargo#7421]: https://github.com/rust-lang/cargo/pull/7421
[cargo#7614]: https://github.com/rust-lang/cargo/issues/7614
[cargo#7699]: https://github.com/rust-lang/cargo/pull/7699
[cargo#7931]: https://github.com/rust-lang/cargo/pull/7931
[cargo#8177]: https://github.com/rust-lang/cargo/pull/8177
[cargo#8490]: https://github.com/rust-lang/cargo/pull/8490
[cargo#8733]: https://github.com/rust-lang/cargo/issues/8733
[cargo#8834]: https://github.com/rust-lang/cargo/pull/8834
[cargo#8945]: https://github.com/rust-lang/cargo/issues/8945
[cargo#8962]: https://github.com/rust-lang/cargo/issues/8962
[cargo#8963]: https://github.com/rust-lang/cargo/issues/8963
[cargo#8968]: https://github.com/rust-lang/cargo/pull/8968
[cargo#9451]: https://github.com/rust-lang/cargo/issues/9451
[cargo#9976]: https://github.com/rust-lang/cargo/issues/9976
[compiler-builtins#411]: https://github.com/rust-lang/compiler-builtins/pull/411
[compiler-builtins#532]: https://github.com/rust-lang/compiler-builtins/pull/532
[compiler-team#343]: https://github.com/rust-lang/compiler-team/issues/343
[internals.r-l.o: Fleshing out libstd scenarios]: https://internals.rust-lang.org/t/fleshing-out-libstd-scenarios/4206
[internals.r-l.o: Refactoring libstd for ultimate portability]: https://internals.rust-lang.org/t/refactoring-std-for-ultimate-portability/4301
[jamesmunns/rfcs#1]: https://github.com/jamesmunns/rfcs/pull/1
[rfcs#1133]: https://github.com/rust-lang/rfcs/pull/1133
[rfcs#1502]: https://github.com/rust-lang/rfcs/pull/1502
[rfcs#1868]: https://github.com/rust-lang/rfcs/pull/1868
[rfcs#2663-t-lang]: https://github.com/rust-lang/lang-team/blob/master/minutes/2019-06-06.md?rgh-link-date=2019-06-06T23%3A20%3A17Z-
[rfcs#2663]: https://github.com/rust-lang/rfcs/pull/2663
[rfcs#3516]: https://rust-lang.github.io/rfcs/3516-public-private-dependencies.html
[rfcs#3716]: https://rust-lang.github.io/rfcs/3716-target-modifiers.html
[rust#107016]: https://github.com/rust-lang/rust/issues/107016
[rust#108924]: https://github.com/rust-lang/rust/pull/108924
[rust#119899]: https://github.com/rust-lang/rust/pull/119899
[rust#120232]: https://github.com/rust-lang/rust/pull/120232
[rust#123360]: https://github.com/rust-lang/rust/pull/123360
[rust#123617]: https://github.com/rust-lang/rust/pull/123617
[rust#128534]: https://github.com/rust-lang/rust/pull/128534
[rust#135395]: https://github.com/rust-lang/rust/pull/135395
[rust#136966]: https://github.com/rust-lang/rust/issues/136966
[rust#31605]: https://github.com/rust-lang/rust/pull/31605
[rust#46439]: https://github.com/rust-lang/rust/pull/46439
[rust#57288]: https://github.com/rust-lang/rust/issues/57288
[rust#64158]: https://github.com/rust-lang/rust/pull/64158
[rust#64316]: https://github.com/rust-lang/rust/pull/64316
[rust#64319]: https://github.com/rust-lang/rust/issues/64319
[rust#67074]: https://github.com/rust-lang/rust/issues/67074
[rust#67650]: https://github.com/rust-lang/rust/issues/67650
[rust#68887]: https://github.com/rust-lang/rust/issues/68887
[rust#69608]: https://github.com/rust-lang/rust/pull/69608
[rust#71009]: https://github.com/rust-lang/rust/pull/71009
[rust#76185]: https://github.com/rust-lang/rust/pull/76185
[rust#77086]: https://github.com/rust-lang/rust/pull/77086
[rust#78790]: https://github.com/rust-lang/rust/pull/78790
[rust#79218]: https://github.com/rust-lang/rust/pull/79218
[rust#79838]: https://github.com/rust-lang/rust/pull/79838
[rust#79958]: https://github.com/rust-lang/rust/pull/79958
[rust#80082]: https://github.com/rust-lang/rust/pull/83655
[rust#83655]: https://github.com/rust-lang/rust/pull/83655
[wg-cargo-std-aware#10]: https://github.com/rust-lang/wg-cargo-std-aware/issues/10
[wg-cargo-std-aware#11]: https://github.com/rust-lang/wg-cargo-std-aware/issues/11
[wg-cargo-std-aware#12]: https://github.com/rust-lang/wg-cargo-std-aware/issues/12
[wg-cargo-std-aware#13]: https://github.com/rust-lang/wg-cargo-std-aware/issues/13
[wg-cargo-std-aware#14-review]: https://github.com/rust-lang/wg-cargo-std-aware/issues/14#issuecomment-2315878717
[wg-cargo-std-aware#14]: https://github.com/rust-lang/wg-cargo-std-aware/issues/14
[wg-cargo-std-aware#15]: https://github.com/rust-lang/wg-cargo-std-aware/issues/15
[wg-cargo-std-aware#16]: https://github.com/rust-lang/wg-cargo-std-aware/issues/16
[wg-cargo-std-aware#17]: https://github.com/rust-lang/wg-cargo-std-aware/issues/17
[wg-cargo-std-aware#18]: https://github.com/rust-lang/wg-cargo-std-aware/issues/18
[wg-cargo-std-aware#19-internals]: https://internals.rust-lang.org/t/dogfooding-z-build-std-in-rustbuild/13775/22
[wg-cargo-std-aware#19]: https://github.com/rust-lang/wg-cargo-std-aware/issues/19
[wg-cargo-std-aware#20]: https://github.com/rust-lang/wg-cargo-std-aware/issues/20
[wg-cargo-std-aware#21]: https://github.com/rust-lang/wg-cargo-std-aware/issues/21
[wg-cargo-std-aware#22]: https://github.com/rust-lang/wg-cargo-std-aware/issues/22
[wg-cargo-std-aware#23]: https://github.com/rust-lang/wg-cargo-std-aware/issues/23
[wg-cargo-std-aware#24]: https://github.com/rust-lang/wg-cargo-std-aware/issues/24
[wg-cargo-std-aware#25]: https://github.com/rust-lang/wg-cargo-std-aware/issues/25
[wg-cargo-std-aware#26]: https://github.com/rust-lang/wg-cargo-std-aware/issues/26
[wg-cargo-std-aware#27]: https://github.com/rust-lang/wg-cargo-std-aware/issues/27
[wg-cargo-std-aware#28]: https://github.com/rust-lang/wg-cargo-std-aware/issues/28
[wg-cargo-std-aware#29]: https://github.com/rust-lang/wg-cargo-std-aware/issues/29
[wg-cargo-std-aware#2]: https://github.com/rust-lang/wg-cargo-std-aware/issues/2
[wg-cargo-std-aware#30]: https://github.com/rust-lang/wg-cargo-std-aware/issues/30
[wg-cargo-std-aware#31]: https://github.com/rust-lang/wg-cargo-std-aware/issues/31
[wg-cargo-std-aware#32]: https://github.com/rust-lang/wg-cargo-std-aware/issues/32
[wg-cargo-std-aware#33]: https://github.com/rust-lang/wg-cargo-std-aware/issues/33
[wg-cargo-std-aware#34]: https://github.com/rust-lang/wg-cargo-std-aware/issues/34
[wg-cargo-std-aware#35]: https://github.com/rust-lang/wg-cargo-std-aware/issues/35
[wg-cargo-std-aware#36]: https://github.com/rust-lang/wg-cargo-std-aware/issues/36
[wg-cargo-std-aware#37]: https://github.com/rust-lang/wg-cargo-std-aware/issues/37
[wg-cargo-std-aware#38]: https://github.com/rust-lang/wg-cargo-std-aware/issues/38
[wg-cargo-std-aware#39]: https://github.com/rust-lang/wg-cargo-std-aware/issues/39
[wg-cargo-std-aware#3]: https://github.com/rust-lang/wg-cargo-std-aware/issues/3
[wg-cargo-std-aware#40]: https://github.com/rust-lang/wg-cargo-std-aware/issues/40
[wg-cargo-std-aware#41]: https://github.com/rust-lang/wg-cargo-std-aware/issues/41
[wg-cargo-std-aware#42]: https://github.com/rust-lang/wg-cargo-std-aware/issues/42
[wg-cargo-std-aware#43-internals]: https://internals.rust-lang.org/t/build-std-and-the-standard-library/11459
[wg-cargo-std-aware#43]: https://github.com/rust-lang/wg-cargo-std-aware/issues/43
[wg-cargo-std-aware#44]: https://github.com/rust-lang/wg-cargo-std-aware/issues/44
[wg-cargo-std-aware#45]: https://github.com/rust-lang/wg-cargo-std-aware/issues/45
[wg-cargo-std-aware#46]: https://github.com/rust-lang/wg-cargo-std-aware/issues/46
[wg-cargo-std-aware#47]: https://github.com/rust-lang/wg-cargo-std-aware/issues/47
[wg-cargo-std-aware#48]: https://github.com/rust-lang/wg-cargo-std-aware/issues/48
[wg-cargo-std-aware#49]: https://github.com/rust-lang/wg-cargo-std-aware/issues/49
[wg-cargo-std-aware#4]: https://github.com/rust-lang/wg-cargo-std-aware/issues/4
[wg-cargo-std-aware#5-internals]: https://internals.rust-lang.org/t/pre-pre-rfc-making-std-dependent-cargo-features-a-first-class-concept/10828
[wg-cargo-std-aware#50]: https://github.com/rust-lang/wg-cargo-std-aware/issues/50
[wg-cargo-std-aware#51]: https://github.com/rust-lang/wg-cargo-std-aware/issues/51
[wg-cargo-std-aware#52]: https://github.com/rust-lang/wg-cargo-std-aware/issues/52
[wg-cargo-std-aware#53]: https://github.com/rust-lang/wg-cargo-std-aware/issues/53
[wg-cargo-std-aware#54]: https://github.com/rust-lang/wg-cargo-std-aware/issues/54
[wg-cargo-std-aware#55]: https://github.com/rust-lang/wg-cargo-std-aware/issues/55
[wg-cargo-std-aware#56]: https://github.com/rust-lang/wg-cargo-std-aware/issues/56
[wg-cargo-std-aware#57]: https://github.com/rust-lang/wg-cargo-std-aware/issues/57
[wg-cargo-std-aware#58]: https://github.com/rust-lang/wg-cargo-std-aware/issues/58
[wg-cargo-std-aware#59]: https://github.com/rust-lang/wg-cargo-std-aware/issues/59
[wg-cargo-std-aware#5]: https://github.com/rust-lang/wg-cargo-std-aware/issues/5
[wg-cargo-std-aware#60]: https://github.com/rust-lang/wg-cargo-std-aware/issues/60
[wg-cargo-std-aware#61]: https://github.com/rust-lang/wg-cargo-std-aware/issues/61
[wg-cargo-std-aware#62]: https://github.com/rust-lang/wg-cargo-std-aware/issues/62
[wg-cargo-std-aware#63]: https://github.com/rust-lang/wg-cargo-std-aware/issues/63
[wg-cargo-std-aware#64]: https://github.com/rust-lang/wg-cargo-std-aware/issues/64
[wg-cargo-std-aware#65]: https://github.com/rust-lang/wg-cargo-std-aware/issues/65
[wg-cargo-std-aware#66]: https://github.com/rust-lang/wg-cargo-std-aware/issues/66
[wg-cargo-std-aware#67]: https://github.com/rust-lang/wg-cargo-std-aware/issues/67
[wg-cargo-std-aware#68]: https://github.com/rust-lang/wg-cargo-std-aware/issues/68
[wg-cargo-std-aware#69]: https://github.com/rust-lang/wg-cargo-std-aware/issues/69
[wg-cargo-std-aware#6]: https://github.com/rust-lang/wg-cargo-std-aware/issues/6
[wg-cargo-std-aware#70]: https://github.com/rust-lang/wg-cargo-std-aware/issues/70
[wg-cargo-std-aware#71]: https://github.com/rust-lang/wg-cargo-std-aware/issues/71
[wg-cargo-std-aware#72]: https://github.com/rust-lang/wg-cargo-std-aware/issues/72
[wg-cargo-std-aware#73]: https://github.com/rust-lang/wg-cargo-std-aware/issues/73
[wg-cargo-std-aware#74]: https://github.com/rust-lang/wg-cargo-std-aware/issues/74
[wg-cargo-std-aware#75]: https://github.com/rust-lang/wg-cargo-std-aware/issues/75
[wg-cargo-std-aware#76]: https://github.com/rust-lang/wg-cargo-std-aware/issues/76
[wg-cargo-std-aware#77]: https://github.com/rust-lang/wg-cargo-std-aware/issues/77
[wg-cargo-std-aware#78]: https://github.com/rust-lang/wg-cargo-std-aware/issues/78
[wg-cargo-std-aware#79]: https://github.com/rust-lang/wg-cargo-std-aware/issues/79
[wg-cargo-std-aware#7]: https://github.com/rust-lang/wg-cargo-std-aware/issues/7
[wg-cargo-std-aware#80]: https://github.com/rust-lang/wg-cargo-std-aware/issues/80
[wg-cargo-std-aware#81]: https://github.com/rust-lang/wg-cargo-std-aware/issues/81
[wg-cargo-std-aware#82]: https://github.com/rust-lang/wg-cargo-std-aware/issues/82
[wg-cargo-std-aware#83]: https://github.com/rust-lang/wg-cargo-std-aware/issues/83
[wg-cargo-std-aware#84]: https://github.com/rust-lang/wg-cargo-std-aware/issues/84
[wg-cargo-std-aware#85]: https://github.com/rust-lang/wg-cargo-std-aware/issues/85
[wg-cargo-std-aware#86]: https://github.com/rust-lang/wg-cargo-std-aware/issues/86
[wg-cargo-std-aware#87]: https://github.com/rust-lang/wg-cargo-std-aware/issues/87
[wg-cargo-std-aware#88]: https://github.com/rust-lang/wg-cargo-std-aware/issues/88
[wg-cargo-std-aware#89]: https://github.com/rust-lang/wg-cargo-std-aware/issues/89
[wg-cargo-std-aware#8]: https://github.com/rust-lang/wg-cargo-std-aware/issues/8
[wg-cargo-std-aware#90]: https://github.com/rust-lang/wg-cargo-std-aware/issues/90
[wg-cargo-std-aware#91]: https://github.com/rust-lang/wg-cargo-std-aware/issues/91
[wg-cargo-std-aware#92]: https://github.com/rust-lang/wg-cargo-std-aware/issues/92
[wg-cargo-std-aware#93]: https://github.com/rust-lang/wg-cargo-std-aware/issues/93
[wg-cargo-std-aware#94]: https://github.com/rust-lang/wg-cargo-std-aware/issues/94
[wg-cargo-std-aware#95]: https://github.com/rust-lang/wg-cargo-std-aware/issues/95

[bootstrap-features-logic]: https://github.com/rust-lang/rust/blob/00b526212bbdd68872d6f964fcc9a14a66c36fd8/src/bootstrap/src/lib.rs#L732
[bootstrap-features-toml]: https://github.com/rust-lang/rust/blob/00b526212bbdd68872d6f964fcc9a14a66c36fd8/bootstrap.example.toml#L816
[bootstrap-sanitizers]: https://github.com/rust-lang/rust/blob/d13a431a6cc69cd65efe7c3eb7808251d6fd7a46/bootstrap.example.toml#L388
[build-std-features]: https://doc.rust-lang.org/cargo/reference/unstable.html#build-std-features
[build-std]: https://doc.rust-lang.org/cargo/reference/unstable.html#build-std
[cargo-docs-registry]: https://doc.rust-lang.org/nightly/nightly-rustc/cargo/sources/registry/index.html
[cargo-docs-renaming]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#renaming-dependencies-in-cargotoml
[cargo-json-schema]: https://doc.rust-lang.org/cargo/reference/registry-index.html#json-schema
[conditional-compilation-config-options]: https://doc.rust-lang.org/reference/conditional-compilation.html#set-configuration-options
[embed-rs-cargo-toml]: https://github.com/embed-rs/stm32f7-discovery/blob/e2bf713263791c028c2a897f2eb1830d7f09eceb/Cargo.toml#L21
[embed-rs-source]: https://github.com/embed-rs/stm32f7-discovery/blob/e2bf713263791c028c2a897f2eb1830d7f09eceb/core/src/lib.rs#L7
[rust-extern-prelude]: https://doc.rust-lang.org/reference/names/preludes.html#extern-prelude
[sgx]: https://github.com/apache/incubator-teaclave-sgx-sdk
[std-build.rs]: https://github.com/rust-lang/rust/blob/f315e6145802e091ff9fceab6db627a4b4ec2b86/library/std/build.rs#L17
[std-unsupported]: https://github.com/rust-lang/rust/blob/f768dc01da9a681716724418ccf64ce55bd396c5/library/std/src/sys/pal/mod.rs#L68-L69
[platform-support]: https://doc.rust-lang.org/nightly/rustc/platform-support.html
[why-not-rustup]: https://hackmd.io/@davidtwco/rkYRlKv_1x

[cargo-add]: https://doc.rust-lang.org/cargo/commands/cargo-add.html
[cargo-bench]: https://doc.rust-lang.org/cargo/commands/cargo-bench.html
[cargo-build]: https://doc.rust-lang.org/cargo/commands/cargo-build.html
[cargo-check]: https://doc.rust-lang.org/cargo/commands/cargo-check.html
[cargo-clean]: https://doc.rust-lang.org/cargo/commands/cargo-clean.html
[cargo-clippy]: https://doc.rust-lang.org/cargo/commands/cargo-clippy.html
[cargo-doc]: https://doc.rust-lang.org/cargo/commands/cargo-doc.html
[cargo-fetch]: https://doc.rust-lang.org/cargo/commands/cargo-fetch.html
[cargo-fix]: https://doc.rust-lang.org/cargo/commands/cargo-fix.html
[cargo-fmt]: https://doc.rust-lang.org/cargo/commands/cargo-fmt.html
[cargo-generate-lockfile]: https://doc.rust-lang.org/cargo/commands/cargo-generate-lockfile.html
[cargo-help]: https://doc.rust-lang.org/cargo/commands/cargo-help.html
[cargo-info]: https://doc.rust-lang.org/cargo/commands/cargo-info.html
[cargo-init]: https://doc.rust-lang.org/cargo/commands/cargo-init.html
[cargo-install]: https://doc.rust-lang.org/cargo/commands/cargo-install.html
[cargo-locate-project]: https://doc.rust-lang.org/cargo/commands/cargo-locate-project.html
[cargo-login]: https://doc.rust-lang.org/cargo/commands/cargo-login.html
[cargo-logout]: https://doc.rust-lang.org/cargo/commands/cargo-login.html
[cargo-metadata]: https://doc.rust-lang.org/cargo/commands/cargo-metadata.html
[cargo-miri]: https://doc.rust-lang.org/cargo/commands/cargo-miri.html
[cargo-new]: https://doc.rust-lang.org/cargo/commands/cargo-new.html
[cargo-owner]: https://doc.rust-lang.org/cargo/commands/cargo-owner.html
[cargo-package]: https://doc.rust-lang.org/cargo/commands/cargo-package.html
[cargo-pkgid]: https://doc.rust-lang.org/cargo/commands/cargo-pkgid.html
[cargo-publish]: https://doc.rust-lang.org/cargo/commands/cargo-publish.html
[cargo-remove]: https://doc.rust-lang.org/cargo/commands/cargo-remove.html
[cargo-report]: https://doc.rust-lang.org/cargo/commands/cargo-report.html
[cargo-run]: https://doc.rust-lang.org/cargo/commands/cargo-run.html
[cargo-rustc]: https://doc.rust-lang.org/cargo/commands/cargo-rustc.html
[cargo-rustdoc]: https://doc.rust-lang.org/cargo/commands/cargo-rustdoc.html
[cargo-search]: https://doc.rust-lang.org/cargo/commands/cargo-search.html
[cargo-test]: https://doc.rust-lang.org/cargo/commands/cargo-test.html
[cargo-tree]: https://doc.rust-lang.org/cargo/commands/cargo-tree.html
[cargo-uninstall]: https://doc.rust-lang.org/cargo/commands/cargo-uninstall.html
[cargo-update]: https://doc.rust-lang.org/cargo/commands/cargo-update.html
[cargo-vendor]: https://doc.rust-lang.org/cargo/commands/cargo-vendor.html
[cargo-version]: https://doc.rust-lang.org/cargo/commands/cargo-version.html
[cargo-yank]: https://doc.rust-lang.org/cargo/commands/cargo-yank.html

[12101111]: https://github.com/12101111
[AZMCode]: https://github.com/AZMCode
[davidtwco]: https://github.com/davidtwco
[Dirbaio]: https://github.com/Dirbaio
[Ericson2314]: https://github.com/Ericson2314
[Gankra]: https://github.com/Gankra
[HenryJk]: https://github.com/HenryJk
[Manishearth]: https://github.com/Manishearth
[Mark-Simulacrum]: https://github.com/Mark-Simulacrum
[MauriceKayser]: https://github.com/MauriceKayser
[SparrowLii]: https://github.com/SparrowLii
[TheBlueMatt]: https://github.com/TheBlueMatt
[adamgemmell]: https://github.com/adamgemmell
[alex]: https://github.com/alex
[alexcrichton]: https://github.com/alexcrichton
[bascule]: https://github.com/bascule
[bjorn3]: https://github.com/bjorn3
[c272]: https://github.com/c272
[catenacyber]: https://github.com/catenacyber
[chaozju]: https://github.com/chaozju
[cr1901]: https://github.com/cr1901
[dullbananas]: https://github.com/dullbananas
[ehuss]: https://github.com/ehuss
[epage]: https://github.com/epage
[errantmind]: https://github.com/errantmind
[fee1-dead]: https://github.com/fee1-dead
[george-hopkins]: https://github.com/george-hopkins
[ghost]: https://github.com/ghost
[harmou01]: https://github.com/harmou01
[hnj2]: https://github.com/hnj2
[illuzen]: https://github.com/illuzen
[jamesmunns]: https://github.com/jamesmunns
[japaric]: https://github.com/japaric
[jdm]: https://github.com/jdm
[joshtriplett]: https://github.com/joshtriplett
[jschwe]: https://github.com/jschwe
[jyn514]: https://github.com/jyn514
[ketsuban]: https://github.com/ketsuban
[madsmtm]: https://github.com/madsmtm
[mati865]: https://github.com/mati865
[mkb2091]: https://github.com/mkb2091
[nagisa]: https://github.com/nagisa
[nazar-pc]: https://github.com/nazar-pc
[parraman]: https://github.com/parraman
[petrochenkov]: https://github.com/petrochenkov
[phip1611]: https://github.com/phip1611
[raoulstrackx]: https://github.com/raoulstrackx
[raphaelcohn]: https://github.com/raphaelcohn
[rust-osdev]: https://github.com/rust-osdev
[saethlin]: https://github.com/saethlin
[skyzh]: https://github.com/skyzh
[tamird]: https://github.com/tamird
[tomassedovic]: https://github.com/tomassedovic
[tmiasko]: https://github.com/tmiasko
[tomaak]: https://github.com/tomaak
[vi]: https://github.com/vi
[wcampbell0x2a]: https://github.com/wcampbell0x2a
[weihanglo]: https://github.com/weihanglo
[wesleywiser]: https://github.com/wesleywiser
[yogh333]: https://github.com/yogh333
