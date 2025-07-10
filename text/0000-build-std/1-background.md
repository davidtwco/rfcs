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
  routines from `compiler-rt` when enabled. It is enabled for the pre-built
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

As per the [Target Tier Policy][target-tier-policy], the Rust project guarantees
one of three levels of support for each built-in target:

- Tier 3 targets exist in the codebase but have no CI support. As a consequence,
  they might not build
- Tier 2 targets have CI checks to ensure the target builds, but they may or may
  not pass tests
- Tier 1 targets have CI checks to ensure that they both build and pass tests

As well as the level of testing a target has before releases, a target's tier
may influence the prioritisation of issues that affect that target.

### std support
[background-std-support]: #std-support

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

### Preludes
[background-preludes]: #preludes

Each Rust crate has a standard library prelude import inserted at the root by
rustc in the form of a glob `use` directive. This brings various commonly-used
items into scope. `std` and `core` each have their own version of the prelude
for each edition. By default the `std` prelude for that edition is imported,
though crates with the `no_std` attribute use the `core` prelude.

rustc also imports the relevant crate (depending on if the `no_std` attribute is
present) into the crate root by injecting an `extern crate core/std` directive.
This is annotated with `macro_use` in order to bring their macros into scope.

#### Extern prelude
[background-extern-prelude]: #extern-prelude

The extern prelude includes crates imported with `extern crate` in the module
or crates passed to rustc via `--extern`. In the 2018 and later editions these
can be referenced directly with `use` statements.

`rustc` also adds `core` to the extern prelude along with `std` if the `no_std`
attribute is not present. `alloc` and `test` are not added to the extern prelude
and so must be brought into scope with an explicit `extern crate` statement.

In order to simulate this behaviour when `-Zbuild-std` passes standard library
dependencies to rustc the `noprelude` option for the `--extern` flag is used.
This avoids crates like `alloc` being added to the extern prelude, but rustc
will still add `core` and `std` to it implicitly as without the use of
`-Zbuild-std`.

## Cargo
[background-cargo]: #cargo

### Registries
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

### Public/private dependencies
[background-pubpriv-dependencies]: #publicprivate-dependencies

[Public and private dependencies][rust#44663] are a currently unstable feature
which enables tracking which dependencies form part of a library's public
interface. This is intended to make it easier to avoid breaking semver
compatibility when developing a library.

With the `public-dependency` feature enabled dependencies are marked as
"private" by default which can be overridden with a `public = true` declaration.

Private dependencies are passed to rustc with an `priv` option to the `--extern`
flag. Dependencies without this option are treated as public by rustc for
compatibility reasons. A new lint is added to rustc,
`exported-private-dependencies`, which is emitted if an item from a private
ependency is exposed as public. In a future edition the lint will be `deny` by
default.

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
