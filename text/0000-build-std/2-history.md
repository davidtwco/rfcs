# History
[history]: #history

*The following summary of the prior art is necessarily less detailed than the
source material, which is exhaustively surveyed in
[Appendix II: Exhaustive literature review][appendix-ii].*

## [rfcs#1133] (2015)
[rfcs-1133-2015]: #rfcs1133-2015

build-std was first proposed in a [2015 RFC (rfcs#1133)][rfcs#1133] by
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
   allowlisted or denylisted to avoid having to address this initially.

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
   or [the relevant section of the literature review for more detail][appendix-ii-impl]

6. **Bugs in the compiler or standard library**
   These aren't especially relevant to this summary, see [the relevant section
   of the literature review for more detail][appendix-ii-bugs]

7. **Cargo feature requests narrowly applied to build-std**
   These aren't especially relevant to this summary, see [the relevant section
   of the literature review for more detail][appendix-ii-cargo-feats]

Since around 2020, activity in the [wg-cargo-std-aware] repository largely
trailed off and there have not been any significant developments related to
build-std since.

### Implementation summary
[implementation-summary]: #implementation-summary

*An exhaustive review of implementation-related issues, pull requests and
discussions can be found in
[the relevant section of the literature review][appendix-ii-impl].*

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

[motivation]: ./3-motivation.md
[appendix-ii]: ./6-appendix-literature-review.md
[appendix-ii-impl]: ./6-appendix-literature-review.md#implementation
[appendix-ii-bugs]: ./6-appendix-literature-review.md#bugs-in-the-compiler-or-standard-library
[appendix-ii-cargo-feats]: ./6-appendix-literature-review.md#cargo-feature-requests-narrowly-applied-to-build-std

[JOSH]: https://josh-project.github.io/josh/intro.html
[rust-lang/cargo]: https://github.com/rust-lang/cargo
[rust-lang/rust]: https://github.com/rust-lang/rust
[wg-cargo-std-aware]: https://github.com/rust-lang/wg-cargo-std-aware
[xargo]: https://github.com/japaric/xargo

[Opaque dependencies]: https://hackmd.io/@epage/ByGfPtRell

[cargo#10129]: https://github.com/rust-lang/cargo/pull/10129
[cargo#14317]: https://github.com/rust-lang/cargo/pull/14317
[cargo#14358]: https://github.com/rust-lang/cargo/pull/14358
[cargo#4959]: https://github.com/rust-lang/cargo/issues/4959
[cargo#7216]: https://github.com/rust-lang/cargo/pull/7216
[cargo#7421]: https://github.com/rust-lang/cargo/pull/7421
[cargo#8490]: https://github.com/rust-lang/cargo/pull/8490
[rfcs#1133]: https://github.com/rust-lang/rfcs/pull/1133
[rfcs#2663]: https://github.com/rust-lang/rfcs/pull/2663
[rfcs#3516]: https://rust-lang.github.io/rfcs/3516-public-private-dependencies.html
[rust#128534]: https://github.com/rust-lang/rust/pull/128534
[rust#67074]: https://github.com/rust-lang/rust/issues/67074
[wg-cargo-std-aware#10]: https://github.com/rust-lang/wg-cargo-std-aware/issues/10
[wg-cargo-std-aware#11]: https://github.com/rust-lang/wg-cargo-std-aware/issues/11
[wg-cargo-std-aware#12]: https://github.com/rust-lang/wg-cargo-std-aware/issues/12
[wg-cargo-std-aware#13]: https://github.com/rust-lang/wg-cargo-std-aware/issues/13
[wg-cargo-std-aware#14-review]: https://github.com/rust-lang/wg-cargo-std-aware/issues/14#issuecomment-2315878717
[wg-cargo-std-aware#14]: https://github.com/rust-lang/wg-cargo-std-aware/issues/14
[wg-cargo-std-aware#15]: https://github.com/rust-lang/wg-cargo-std-aware/issues/15
[wg-cargo-std-aware#16]: https://github.com/rust-lang/wg-cargo-std-aware/issues/16
[wg-cargo-std-aware#17]: https://github.com/rust-lang/wg-cargo-std-aware/issues/17
[wg-cargo-std-aware#19]: https://github.com/rust-lang/wg-cargo-std-aware/issues/19
[wg-cargo-std-aware#20]: https://github.com/rust-lang/wg-cargo-std-aware/issues/20
[wg-cargo-std-aware#21]: https://github.com/rust-lang/wg-cargo-std-aware/issues/21
[wg-cargo-std-aware#22]: https://github.com/rust-lang/wg-cargo-std-aware/issues/22
[wg-cargo-std-aware#23]: https://github.com/rust-lang/wg-cargo-std-aware/issues/23
[wg-cargo-std-aware#24]: https://github.com/rust-lang/wg-cargo-std-aware/issues/24
[wg-cargo-std-aware#25]: https://github.com/rust-lang/wg-cargo-std-aware/issues/25
[wg-cargo-std-aware#26]: https://github.com/rust-lang/wg-cargo-std-aware/issues/26
[wg-cargo-std-aware#29]: https://github.com/rust-lang/wg-cargo-std-aware/issues/29
[wg-cargo-std-aware#2]: https://github.com/rust-lang/wg-cargo-std-aware/issues/2
[wg-cargo-std-aware#30]: https://github.com/rust-lang/wg-cargo-std-aware/issues/30
[wg-cargo-std-aware#36]: https://github.com/rust-lang/wg-cargo-std-aware/issues/36
[wg-cargo-std-aware#38]: https://github.com/rust-lang/wg-cargo-std-aware/issues/38
[wg-cargo-std-aware#39]: https://github.com/rust-lang/wg-cargo-std-aware/issues/39
[wg-cargo-std-aware#3]: https://github.com/rust-lang/wg-cargo-std-aware/issues/3
[wg-cargo-std-aware#40]: https://github.com/rust-lang/wg-cargo-std-aware/issues/40
[wg-cargo-std-aware#43]: https://github.com/rust-lang/wg-cargo-std-aware/issues/43
[wg-cargo-std-aware#45]: https://github.com/rust-lang/wg-cargo-std-aware/issues/45
[wg-cargo-std-aware#46]: https://github.com/rust-lang/wg-cargo-std-aware/issues/46
[wg-cargo-std-aware#47]: https://github.com/rust-lang/wg-cargo-std-aware/issues/47
[wg-cargo-std-aware#4]: https://github.com/rust-lang/wg-cargo-std-aware/issues/4
[wg-cargo-std-aware#50]: https://github.com/rust-lang/wg-cargo-std-aware/issues/50
[wg-cargo-std-aware#51]: https://github.com/rust-lang/wg-cargo-std-aware/issues/51
[wg-cargo-std-aware#5]: https://github.com/rust-lang/wg-cargo-std-aware/issues/5
[wg-cargo-std-aware#64]: https://github.com/rust-lang/wg-cargo-std-aware/issues/64
[wg-cargo-std-aware#68]: https://github.com/rust-lang/wg-cargo-std-aware/issues/68
[wg-cargo-std-aware#69]: https://github.com/rust-lang/wg-cargo-std-aware/issues/69
[wg-cargo-std-aware#6]: https://github.com/rust-lang/wg-cargo-std-aware/issues/6
[wg-cargo-std-aware#7]: https://github.com/rust-lang/wg-cargo-std-aware/issues/7
[wg-cargo-std-aware#85]: https://github.com/rust-lang/wg-cargo-std-aware/issues/85
[wg-cargo-std-aware#86]: https://github.com/rust-lang/wg-cargo-std-aware/issues/86
[wg-cargo-std-aware#87]: https://github.com/rust-lang/wg-cargo-std-aware/issues/87
[wg-cargo-std-aware#88]: https://github.com/rust-lang/wg-cargo-std-aware/issues/88
[wg-cargo-std-aware#8]: https://github.com/rust-lang/wg-cargo-std-aware/issues/8
[wg-cargo-std-aware#90]: https://github.com/rust-lang/wg-cargo-std-aware/issues/90
[wg-cargo-std-aware#92]: https://github.com/rust-lang/wg-cargo-std-aware/issues/92

[build-std-features]: https://doc.rust-lang.org/cargo/reference/unstable.html#build-std-features
[build-std]: https://doc.rust-lang.org/cargo/reference/unstable.html#build-std
[std-build.rs]: https://github.com/rust-lang/rust/blob/f315e6145802e091ff9fceab6db627a4b4ec2b86/library/std/build.rs#L17
[std-unsupported]: https://github.com/rust-lang/rust/blob/f768dc01da9a681716724418ccf64ce55bd396c5/library/std/src/sys/pal/mod.rs#L68-L69

[Ericson2314]: https://github.com/Ericson2314
[epage]: https://github.com/epage