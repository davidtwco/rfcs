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
- The standard library has and has had dependencies which require a more
  complicated build environment than typical Rust projects
  - e.g. requiring a working C toolchain to build `compiler_builtins`' `c`
    feature
- To varying degrees at different times in its development, the standard
  library's implementation has been tied to the compiler implementation and has had
  to change in lockstep

Not all targets have a pre-built standard library distributed via rustup, though
it is a minimum requirement for certain platform support tiers. According to
rustc's [platform support docs][platform-support], for tier three targets:

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

As an innate property of the target, not all targets can support the `std` crate
This is independent of its tier, where as stated in the
[Target Tier Policy][target-tier-policy] lower-tier targets may not have a
complete implementation for all APIs in the crates they can support.

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

Cargo supports explicitly declaring a dependency on crates with the same names
as standard library crates with a `path` source
(e.g. `core = { path = "../my_core" }`), which rustc will load instead of crates
in the sysroot. Crates with these dependencies are not accepted by crates.io,
but there are crates on GitHub that use this pattern, such as
[embed-rs/stm32f7-discovery][embed-rs-cargo-toml], which are used as `git`
dependencies of other crates on GitHub.

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
on `backtrace-sys` but now uses `gimli` ([rust#46439]), a pure-rust
implementation. There are still some C dependencies:

- `libunwind` will either link to the LLVM `libunwind` or the system's
  `libunwind`/`libgcc_s`. LLVM's `libunwind` is shipped as part of the
  rustup component for the standard library and will be linked against
  when `-Clink-self-contained` is used
  - This only applies to Linux and Fuchsia targets
- `compiler_builtins` has an optional `c` feature that will use optimised
  routines from `compiler-rt` when enabled. It is enabled for the pre-built
  standard library
- `compiler_builtins` has an optional `mem` feature that provides symbols
  for common memory routines (e.g. `memcpy`)
  - It is enabled automatically on `no_std` platforms as when `std` is built
    `libc` provides these routines.
  - Users often rely on weak linkage to override these symbols when required,
    but in scenarios where weak linkage is not supported users must directly
    turn the feature off.
- To use sanitizers, the sanitizer runtimes from LLVM's compiler-rt need to
  be linked against. Building of these is enabled in `bootstrap.toml`
  ([`build.sanitizers`][bootstrap-sanitizers]) and they are
  included in the rustup components shipped by the project.

Dependencies of the standard library may use unstable or internal compiler and
language features only when they are a dependency of the standard library.

### Features
[background-features]: #features

There are a handful of features defined in the standard library crates'
`Cargo.toml`s. These features are not strictly additive (`llvm-libunwind` and
`system-llvm-libunwind` are mutually exclusive). There is currently no stable
existing mechanism for users to enable or disable these features. The default
set of features is determined by [logic in bootstrap][bootstrap-features-logic]
and [the `rust.std-features` key in `bootstrap.toml`][bootstrap-features-toml].
The enabled features are often different depending on the target.

It is also common for user crates to depend on the standard library (via
`#![no_std]`) conditional on Cargo features being enabled or disabled (e.g. a
`std` feature or if `--test` is used).

### Target support
[background-target-support]: #target-support

The `std` crate's [`build.rs`][std-build.rs] checks for supported values of the
`CARGO_CFG_TARGET_*` environment variables. These variables are akin to the
conditional compilation [configuration options][conditional-compilation-config-options],
and often correspond to parts of the target triple (for example,
`CARGO_CFG_TARGET_OS` corresponds to the "os" part of a target triple - "linux"
in "aarch64-unknown-linux-gnu"). This filtering is strict enough to distinguish
between built-in targets but loose enough to match similar custom targets. There
is no equivalent mechanism on the `alloc` or `core` crates.

When encountering an unknown or unsupported operating system then the
`restricted_std` cfg is set. `restricted_std` marks the entire standard library
as unstable, requiring `feature(restricted_std)` to be enabled on any crate that
depends on it. The only way for users to enable the `restricted_std` feature on
behalf of dependencies is the uncommon `-Zcrate-attr=features(restricted_std)`
rustc flag and users commonly report that they are not aware how to do this.

Cargo and rustc support custom targets, defined in JSON files according to an
unstable schema defined in the compiler. On nightly, users can dump the
target-spec-json for an existing target using `--print target-spec-json`. This
JSON can be saved in a file, tweaked and used as the argument to `--target`. It
is unintentional but custom target specifications can be used with `--target`
even on stable toolchains ([rust#71009] proposes destabilising this behaviour).
However, as custom targets do not have a pre-built standard library and so must
use `-Zbuild-std`, their use is relegated to nightly toolchains in practice.
Custom targets may have `restricted_std` set depending on their `cfg`
configuration options.

## Prelude
[background-prelude]: #prelude

rustc has the concept of the "extern prelude" which is effectively the set of
crates that can be referred to without an explicit `extern crate` statement.
Originally this was populated by users writing `extern crate $crate` in their
code for each direct dependency. Since the 2018 edition, crates passed via
`--extern` added to the extern prelude. `core` is always added to the extern
prelude. For crates without `#![no_std]`, `std` is added to the extern prelude.

The `core` or `std` prelude is added (depending on the presence of `#![no_std]`)
in the form of a `use $crate::prelude::rust_20XX::*` statement.

`extern crate` can still be used and will search for the dependency in locations
where direct dependencies can be found, such as `-L crate=` paths or in the
sysroot. `-L dependency=` paths will not be searched, as these directories only
contain indirect dependencies (i.e. dependencies of direct dependencies).

Although only `std` or `core` are added to the extern prelude automatically,
users can still write `extern crate alloc` or `extern crate test` to load them
from the sysroot.

`--extern` has a `noprelude` modifier which will allow the user to use
`--extern` to specify the location at which a crate can be found without adding
it to the extern prelude. This could allow a path for crates like `alloc` or
`test` to be provided without effecting the observable behaviour of the
language.

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

`std` is also a panic handler. `std`'s panic handler function and its
`std::panic!` macro print panic information to stderr and delegate to a
*panic runtime* to decide what to do next, determined by the *panic strategy*.

There are two panic runtime crates in the standard library - `panic_unwind`
(which gracefully unwinds the stack using `libunwind` and performs cleanup) and
`panic_abort` (which terminates the program shortly after being called). Each
target supported by rustc specifies a default panic strategy - either "unwind"
or "abort" - though these are only relevant if `std`'s panic handler is used
(i.e. the target isn't a `no_std` target or being used with a `no_std` crate).

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
panic handler, which can dramatically reduce code size. `std` and `alloc` have
the same feature which enable the feature in `core`. `std`'s feature also adds
an immediate abort to its `panic!` macro.

## Cargo
[background-cargo]: #cargo

Cargo's building of the dependency graph is largely driven by the registry
index, except for crates from `git` or `path` sources.

[Cargo registries][cargo-docs-registry], like crates.io, are centralised sources
for crates. A registry's index is the interface between Cargo and the registry
that Cargo queries to know which versions are available for any given crate,
what its dependencies are, etc.

Cargo can query registries using a Git protocol which caches the registry on
disk, or using a sparse protocol which exposes the index over HTTP and allows
Cargo to avoid Cargo having a local copy of the whole index, which has become
quite large for crates.io.

crates.io's registry index is exposed as both a HTTP API and a Git repository -
[rust-lang/crates.io-index] - both are updated automatically by crates.io when
crates are published, yanked, etc. The HTTP API is mostly used.

Each crate in the registry index has a JSON file, following
[a defined schema][cargo-json-schema] which is jointly maintained by the Cargo
and crates.io teams. Crates may refer to those in other registries, but all
non-`path`/`git` crates in the dependency graph must exist in a registry. As the
registry index drives the building of Cargo's dependency graph, all
non-`path`/`git` crates that end up in the dependency graph must be present a
registry.

When a package is published, Cargo posts a JSON blob to the registry which is
not a index entry but has sufficient information to generate one. crates.io does
not use Cargo's JSON blob, instead re-generating it from the `Cargo.toml` (this
avoids the index and `Cargo.toml` from going out-of-sync due to bugs or
malicious publishes). As a consequence, changes to the index format must be
duplicated in Cargo and crates.io. Behind the scenes, data from the `Cargo.toml`
extracted by crates.io is written to a database, which is where the index entry
and frontend are generated from.

Dependency information of crates in the registry are rendered in the crates.io
frontend.

Registries can have different policies for what crates are accepted. For
example, crates.io does not permit publishing packages named `std` or `core` but
other registries might.

### Public/private dependencies
[background-pubpriv-dependencies]: #publicprivate-dependencies

[Public and private dependencies][rust#44663] are an unstable feature which
enables declaring which dependencies form part of a library's public interface,
so as to make it easier to avoid breaking semver compatibility.

With the `public-dependency` feature enabled, dependencies are marked as
"private" by default which can be overridden with a `public = true` declaration.

Private dependencies are passed to rustc with an `priv` modifier to the
`--extern` flag. Dependencies without this modifier are treated as public by
rustc for backwards compatibility reasons. rust emits the
`exported-private-dependencies` lint if an item from a private dependency is
re-exported.

## Target modifiers
[background-target-modifiers]: #target-modifiers

[rfcs#3716] introduced the concept of *target modifiers* to rustc. Flags marked
as target modifiers must match across the entire crate graph or the compilation
will fail.

For example, flags are made target modifiers when they change the ABI of
generated code and could result in unsound ABI mismatches if two crates are
linked together with different values of the flag set.

[implementation-summary]: ./2-history.md#implementation-summary

[JOSH]: https://josh-project.github.io/josh/intro.html
[panic-abort]: https://crates.io/crates/panic-abort
[panic-halt]: https://crates.io/crates/panic-halt
[panic-itm]: https://crates.io/crates/panic-itm
[panic-semihosting]: https://crates.io/crates/panic-semihosting
[rust-lang/crates.io-index]: https://github.com/rust-lang/crates.io-index
[rust-lang/rust]: https://github.com/rust-lang/rust

[rfcs#3716]: https://rust-lang.github.io/rfcs/3716-target-modifiers.html
[rust#46439]: https://github.com/rust-lang/rust/pull/46439
[rust#44663]: https://github.com/rust-lang/rust/issues/44663
[rust#71009]: https://github.com/rust-lang/rust/issues/71009

[bootstrap-features-logic]: https://github.com/rust-lang/rust/blob/00b526212bbdd68872d6f964fcc9a14a66c36fd8/src/bootstrap/src/lib.rs#L732
[bootstrap-features-toml]: https://github.com/rust-lang/rust/blob/00b526212bbdd68872d6f964fcc9a14a66c36fd8/bootstrap.example.toml#L816
[bootstrap-sanitizers]: https://github.com/rust-lang/rust/blob/d13a431a6cc69cd65efe7c3eb7808251d6fd7a46/bootstrap.example.toml#L388
[cargo-docs-registry]: https://doc.rust-lang.org/nightly/nightly-rustc/cargo/sources/registry/index.html
[cargo-json-schema]: https://doc.rust-lang.org/cargo/reference/registry-index.html#json-schema
[conditional-compilation-config-options]: https://doc.rust-lang.org/reference/conditional-compilation.html#set-configuration-options
[embed-rs-cargo-toml]: https://github.com/embed-rs/stm32f7-discovery/blob/e2bf713263791c028c2a897f2eb1830d7f09eceb/Cargo.toml#L21
[target-tier-policy]: https://doc.rust-lang.org/nightly/rustc/target-tier-policy.html
[std-build.rs]: https://github.com/rust-lang/rust/blob/f315e6145802e091ff9fceab6db627a4b4ec2b86/library/std/build.rs#L17
[platform-support]: https://doc.rust-lang.org/nightly/rustc/platform-support.html
