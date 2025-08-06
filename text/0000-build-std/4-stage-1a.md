# Proposal
[proposal]: #proposal

Cargo configuration will contain a new key `build-std` under the `[build]`
section ([?][rationale-build-std-in-config]), permitting one of two values -
"off" ([?][rationale-build-std-off]) or "always", defaulting to "off":

```toml
[build]
build-std = "always" # or `off`
```

`build-std` can also be specified in the `[target.<triple>]` and
`[target.<cfg>]` sections ([?][rationale-build-std-target-section]):

```toml
[target.aarch64-unknown-illumos]
build-std = "always" # or `off`
```

The `build-std` configuration locations have the following precedence
([?][rationale-build-std-precedence]):

1. `[target.<triple>]`
2. `[target.<cfg>]`
3. `[build]`

As the Cargo configuration is local to the current installation of Cargo
(typically in `~/.config/cargo`), the value of `build-std` is not influenced by
the dependencies of the current crate.

When `build-std` is set to "always", then the standard library will be
unconditionally recompiled ([?][rationale-unconditional]) in its release profile
as part of every clean build ([?][rationale-release-profile]). This is primarily
useful for users of tier three targets.

> [!NOTE]
>
> Configuration of the pre-built standard library is split across bootstrap and
> the Cargo packages for the standard library. As much of this configuration as
> possible should be moved to the Cargo profile for these packages so that the
> artifacts produced by build-std match the pre-built standard library as much
> as is feasible.

Alongside `build-std`, a `build-std-crate` key will be introduced
([?][rationale-build-std-crate]), which can be used to specify which crates from
the standard library is to be built. Only "core", "alloc" and "std" are valid
values for `build-std-crate`.

```toml
[build]
build-std-crate = "std"
```

If [*Stage 1b* of this proposal][stage1b] is implemented then `build-std-crate`
will not be used unless explicitly set and the crate graph's dependencies on the
standard library will determine which crates are built instead. Otherwise,
`build-std-crate` will default to "std".

If `std` is to be built and Cargo is building a test using the default test
harness then Cargo will also build the `test` crate.

> [!NOTE]
>
> Inspired by the concept of [opaque dependencies][Opaque dependencies], the
> standard library is resolved differently to other dependencies:
>
> - The lockfile included in the standard library source will be used when
>   resolving the standard library's dependencies ([?][rationale-lockfile]).
>
> - The dependencies of the standard library crates are entirely opaque to the
>   user. Different semver-compatible versions of these dependencies can
>   exist in the user's resolve. The user cannot control compilation any of
>   the dependencies of the `core`, `alloc` or `std` standard library crates
>   individually (via profile overrides, for example).
>
> - The profile defined by the standard library will be used.
>
> Cargo will resolves the dependencies of opaque dependencies, such as the
> standard library, separately in their own workspaces. The root of such a
> resolve will be the crate specified in `build-std-crates`, or, if stage 1b is
> implemented, the unified set of packages that any crate in the dependency has
> a direct dependency on. A dependency on the relevant roots are added to all
> crates in the "parent" resolve.
>
> Regardless of which standard library crates are being built, Cargo will build
> the `sysroot` crate of the standard library workspace. `alloc` and `std` will
> be optional dependencies of the `sysroot` crate which will be enabled when the
> user has requested them. Panic runtimes are dependencies of `std` and will be
> enabled depending on the features that Cargo passes to `std` (see
> [*Panic strategies*][panic-strategies]).
>
> rustc loads panic runtimes in a different way to most dependencies, and
> without looking in the sysroot they will fail to load correctly unless passed
> in with `--extern`. rustc will need to be patched to be able to load panic
> runtimes from `-L dependency=` paths in line with other transitive
> dependencies.
>
> The standard library will always be a non-incremental build
> ([?][rationale-incremental]), with no `depinfo` produced, and only a `rlib`
> produced (no `dylib`) ([?][rationale-no-dylib]). It will be built in the Cargo
> `target` directory of the crate or workspace like any other dependency.

The host pre-built standard library will always be used for procedural macros
and build scripts ([?][rationale-sysroot-for-host-deps]). Multi-target projects
(resulting from the `target` field in artifact dependencies or the use of
`per-pkg-target` fields) may result in the standard library being built multiple
times - once for each target in the project.

*See the following sections for rationale/alternatives:*

- [*Why put `build-std` in the Cargo config?*][rationale-build-std-in-config]
- [*Why accept `off` as a value for `build-std`?*][rationale-build-std-off]
- [*Why add `build-std` to the `[target.<triple>]` and `[target.<cfg>]` sections?*][rationale-build-std-target-section]
- [*Why does `[target]` take precedence over `[build]` for `build-std`?*][rationale-build-std-precedence]
- [*Why does "always" rebuild unconditionally?*][rationale-unconditional]
- [*Why does "always" rebuild in release profile?*][rationale-release-profile]
- [*Why add `build-std-crate`?*][rationale-build-std-crate]
- [*Why use the lockfile of the `rust-src` component?*][rationale-lockfile]
- [*Why not build the standard library in incremental?*][rationale-incremental]
- [*Why not produce a `dylib` for the standard library?*][rationale-no-dylib]
- [*Why use the pre-built standard library for procedural macros and build-scripts?*][rationale-sysroot-for-host-deps]

*See the following sections for relevant unresolved questions:*

- [*What should the `build-std` configuration in `.cargo/config` be named?*][unresolved-config-name]
- [*What should the "always" and "off" values of `build-std` be named?*][unresolved-config-values]
- [*What should `build-std-crate` be named?*][unresolved-build-std-crate-name]

## Interactions with `#![no_std]`
[interactions-with-no_std]: #interactions-with-no_std

Behaviour of crates using `#![no_std]` will not change whether or not `std` is
rebuilt and passed via `--extern` to rustc, and `#![no_std]` will still be
required in order for `rustc` to not attempt to load `std` and add it to the
extern prelude.

*See the following sections for rationale/alternatives:*

- [*Why not replace `#![no_std]` as the source-of-truth for whether a crate depends on `std`?*][rationale-replace-no_std]

## `restricted_std`
[restricted_std]: #restricted_std

The existing `restricted_std` mechanism will be removed from the standard
library's [`build.rs`][std-build.rs].

*See the following sections for rationale/alternatives:*

- [*Why remove `restricted_std`?*][rationale-remove-restricted-std]

## Custom targets
[custom-targets]: #custom-targets

Cargo will detect when the standard library is to be built for a custom target
and will emit an error ([?][rationale-disallow-custom-targets]).

> [!NOTE]
>
> Cargo could detect use of a custom target either by comparing it with the list
> of built-in targets that rustc reports knowing about (via `--print target-list`)
> or by checking if a file exists at the path matching the provided target name.
>
> This does not require any changes to rustc. If it is invoked to build the
> standard library then it will continue to do so, as is possible today, it is
> only the build-std functionality in Cargo that will not support custom targets
> initially.

Custom targets can still be used with build-std on nightly toolchains provided
that `-Zunstable-options` is provided to Cargo.

*See the following sections for rationale/alternatives:*

- [*Why disallow custom targets?*][rationale-disallow-custom-targets]

*See the following sections for future possibilities:*

- [*Allow custom targets with build-std*][future-custom-targets]

## Preventing implicit sysroot dependencies
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

## Vendored `rust-src`
[vendored-rust-src]: #vendored-rust-src

When it is necessary to build the standard library, Cargo will look for sources
in a fixed location in the sysroot ([?][rationale-custom-src-path]):
`lib/rustlib/src`. rustup's `rust-src` component downloads standard library
sources to this location and will be made a default component. If the sources
are not found, Cargo will emit an error and recommend the user download
`rust-src` if using rustup.

`rust-src` will contain the sources for the standard library crates as well as
its vendored dependencies ([?][rationale-vendoring]). As a consequence sources
of standard library dependencies will not need be fetched from crates.io.

> [!NOTE]
>
> Cargo will not perform any checks to ensure that the sources in `rust-src`
> have been modified ([?][rationale-src-modifications]). It will be documented
> that modifying these sources is not supported.

*See the following sections for rationale/alternatives:*

- [*Why not allow the source path for the standard library be customised?*][rationale-custom-src-path]
- [*Why vendor standard library dependencies?*][rationale-vendoring]
- [*Why not check if `rust-src` has been modified?*][rationale-src-modifications]

## Panic strategies
[panic-strategies]: #panic-strategies

Panic strategies are unlike other profile settings insofar as they influence
which crates are built and which flags are passed to the standard library build.
For example, if `panic = "unwind"` were set in the Cargo profile then the
`panic_unwind` feature would need to be provided to `std` and `-Cpanic=unwind`
passed to suggest that the compiler use that panic runtime.

If Cargo is not building `std`, then neither of the panic runtimes will be
built. In this circumstance rustc will continue to throw an error when a
unwinding panic strategy is chosen.

If the Cargo would build `std` for a project then Cargo's behaviour depends on
whether or not `panic` is set in the profile:

- If `panic` is not set in the profile then unwinding may still be the default
  for the target and Cargo will need to enable the `panic_unwind` feature to the
  `sysroot` crate to build `panic_unwind` just in case it is used

- If `panic` is set to "unwind" then the `panic_unwind` feature of `sysroot`
  will be enabled and `-Cpanic=unwind` will be passed

- If `panic` is set to "abort" then `-Cpanic=abort` will be passed

  - `panic_abort` is a non-optional dependency of `std` so it will always be
    built

Tests, benchmarks, build scripts and proc macros continue to ignore the "panic"
setting and `panic = "unwind"` is always used - which means the standard library
needs to be recompiled again if the user is using "abort". Once
`panic-abort-tests` is stabilised, the standard library can be built with the
profile's panic strategy even for tests and benchmarks.

In line with Cargo's stance on not parsing the `RUSTFLAGS` environment variable,
it will not be checked for compilation flags that would require additional
crates to be built for compilation to succeed.

> [!NOTE]
>
> The `unwind` crate will continue to link to the system's `libunwind` which
> will need to match the target modifiers used by the standard library to avoid
> incompatibilities. Likewise, if `llvm-libunwind`, `-Clink-self-contained=yes`
> or `-Ctarget-feature=+crt-static` are used and the distributed `libunwind` is
> used then it will also need to match the target modifiers of the standard
> library to avoid incompatibilities.

*See the following sections for future possibilities:*

- [*Avoid building `panic_unwind` unnecessarily*][future-panic_unwind]

## Building the standard library on a stable toolchain
[building-the-standard-library-on-a-stable-toolchain]: #building-the-standard-library-on-a-stable-toolchain

rustc will automatically assume `RUSTC_BOOTSTRAP` when the source path of the
crate being compiled is within the same sysroot as the rustc binary being
invoked ([?][rationale-implied-bootstrap]). Cargo will not need to use
`RUSTC_BOOTSTRAP` when compiling the standard library with a stable toolchain.

*See the following sections for rationale/alternatives:*

- [*Why allow building from the sysroot with implied `RUSTC_BOOTSTRAP`?*][rationale-implied-bootstrap]

## Self-contained objects
[self-contained-objects]: #self-contained-objects

A handful of targets require linking against special object files, such as
`windows-gnu`, `linux-musl` and `wasi` targets. For example, `linux-musl`
targets require `crt1.o`, `crti.o`, `crtn.o`, etc.

Since [rust#76158]/[compiler-team#343], the compiler has a stable
`-Clink-self-contained` flag which will look for special object files in
expected locations, typically populated by the `rust-std` components. Its
behaviour can be forced by `-Clink-self-contained=true`, but is force-enabled
for some targets and inferred for others.

Rust can start to ship `rust-self-contained` components for any targets which
need it. These components will contain the special object files normally
included in `rust-std`, and will be distributed for all tiers of targets. While
generally these objects are specific to the architecture and C runtime (CRT)
(and so `rust-self-contained-$arch-$crt` could be sufficient and result in fewer
overall components), it's technically possible that Rust could support two
targets with the same architecture and same CRT but different versions of the
CRT, so having target-specific components is most future-proof. These would
replace the `self-contained` directory in existing `rust-std` components.

Similarly, for any architectures which require it, LLVM's `libunwind` will be
built and shipped in the `rust-self-contained` component.

As long as these components have been downloaded, as well as any other support
components, such as `rust-mingw`, rustc's `-Clink-self-contained` will be able
to link against the object files and build-std should never fail on account of
missing special object files.

*See the following sections for future possibilities:*

- [*Enable local recompilation of special object files/sanitizer runtimes*][future-recompile-special]

## `compiler-builtins`
[compiler-builtins]: #compiler-builtins

`compiler-builtins` is always built with `-Ccodegen-units=10000` to force each
intrinsic into its own object file to avoid symbol clashes with libgcc. This is
currently enforced with a profile override in the standard library's workspace.

rustc will automatically use a large number of codegen units for the
`compiler-builtins` crate, unless manually specified using the `-Ccodegen-units`
flag (to support users, like Rust for Linux, that prefer a single codegen unit).
This prevents `compiler-builtins` from having to be special-cased in the
standard library workspace.

> [!NOTE]
>
> [rust#135395] could be resurrected to implement this.

### `compiler-builtins/mem`
[compiler-builtins-mem]: #compiler-builtinsmem

The `mem` feature of `compiler_builtins` (and the subsequent
`compiler-builtins-mem` feature of `core`, `alloc`, `std` which forward to
`compiler_builtins/mem`) is required by `no_std` crates as a `std` dependency
will not be providing these symbols through its dependency on `libc`.

It is necessary that the `compiler-builtins-mem` feature of `alloc` and/or
`core` be enabled when `libc` is not in the crate graph
([?][rationale-no-weak-linkage]).

*See the following sections for rationale/alternatives:*

- [*Why not use weak linkage for `compiler-builtins/mem` symbols?*][rationale-no-weak-linkage]

### `compiler-builtins/c`
[compiler-builtins-c]: #compiler-builtinsc

The [`c` feature][background-dependencies] of `compiler_builtins` (which is also
exposed by `core`, `alloc` and `std` through `compiler-builtins-c`) causes its
`build.rs` file to build and link in more optimised C versions of intrinsics.

It will not be enabled by default because it is possible that the target
platform does not have a suitable C compiler available. The user being able to
enable this manually will be enabled through work on features (see
[*Allow enabling/disabling features with build-std*][future-features] from Stage
1b). Once the user can enable `compiler-builtins/c`, they will need to manually
configure `CFLAGS` to ensure that the C components will link with Rust code.

## Caching
[caching]: #caching

Standard library artifacts built by build-std will not be shared between crates
or workspaces, as they only exist in Cargo's target directory for a specific
crate or workspace ([?][rationale-caching]).

*See the following sections for rationale/alternatives:*

- [*Why not globally cache builds of the standard library?*][rationale-caching]

## Generated documentation
[generated-documentation]: #generated-documentation

When running `cargo doc` for a project to generate documentation and rebuilding
the standard library, the generated documentation for the user's crates will
link to the locally generated documentation for the `core`, `alloc` and `std`
crates, rather than the upstream hosted generation as is typical for non-locally
built standard libraries.

*See the following sections for rationale/alternatives:*

- [*Why not link to hosted standard library documentation in generated docs?*][rationale-generated-docs]

## Cargo subcommands
[cargo-subcommands]: #cargo-subcommands

As opaque dependencies, any Cargo command which accepts a package spec with `-p`
will only additionally recognise `core`, `alloc` and `std` and none of their
dependencies. Many of Cargo's subcommands will need modification to support
build-std:

[`cargo clean`][cargo-clean] will additionally delete any builds of the standard
library performed by build-std.

[`cargo fetch`][cargo-fetch] will not fetch the standard library dependencies as
they are already vendored in the `rust-src` component.

[`cargo miri`][cargo-miri] is not built into Cargo, it is shipped by miri, but
is mentioned in Cargo's documentation. `cargo miri` is unchanged by this RFC,
but build-std is one step towards `cargo miri` requiring less special support.

> [!NOTE]
>
> `cargo miri` could be re-implemented using build-std to enable a `miri`
> profile and always rebuild. The `miri` profile would be configured in the
> standard library's workspace, setting the flags/options necessary for `miri`.

[`cargo report`][cargo-report] will not include reports from the standard
library crates or their dependencies.

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

This stage has no implications for the following Cargo subcommands:

- [`cargo add`][cargo-add]
- [`cargo remove`][cargo-remove]
- [`cargo fmt`][cargo-fmt]
- [`cargo generate-lockfile`][cargo-generate-lockfile]
- [`cargo help`][cargo-help]
- [`cargo info`][cargo-info]
- [`cargo init`][cargo-init]
- [`cargo install`][cargo-install]
- [`cargo locate-project`][cargo-locate-project]
- [`cargo login`][cargo-login]
- [`cargo logout`][cargo-logout]
- [`cargo metadata`][cargo-metadata]
- [`cargo new`][cargo-new]
- [`cargo owner`][cargo-owner]
- [`cargo package`][cargo-package]
- [`cargo pkgid`][cargo-pkgid]
- [`cargo publish`][cargo-publish]
- [`cargo search`][cargo-search]
- [`cargo tree`][cargo-tree]
- [`cargo uninstall`][cargo-uninstall]
- [`cargo version`][cargo-version]
- [`cargo yank`][cargo-yank]

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This section aims to justify all of the decisions made in the proposed design
from [*Proposal*][proposal] and discuss why alternatives were not chosen.

## Why put `build-std` in the Cargo config?
[rationale-build-std-in-config]: #why-put-build-std-in-the-cargo-config

There are various alternatives to putting `build-std` in the Cargo configuration:

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

2. build-std could be enabled or disabled in the `Cargo.toml`. However, under
   which conditions the standard library is rebuilt is better determined by the
   user of Cargo, rather than the package being built.

   A user may want to never rebuild the standard library so as to avoid
   invalidating the guarantees of their qualified toolchain, or may want to
   rebuild unconditionally to further optimise the standard library for their
   known deployment platform, or may only want to rebuild as necessary to ensure
   the build will succeed. All of these rationale can apply to the same crate in
   different circumstances, so it doesn't make sense for a crate to decide this
   once in its `Cargo.toml`.

   It would be a waste of resources if a dependency declared that it must always
   rebuild the standard library when the pre-built crate would be sufficient and
   this could not be overridden. It is also unclear how to aggregate different
   configurations of the `build-std` key from different crates in the dependency
   graph into a single value.

While using `build-std` key in the Cargo configuration shares some of the
downsides of using an explicit flag - not having a natural extension point for
other Cargo options exposed to dependencies - [Stage 1b][stage1b] addresses
these concerns.

↩ [*Proposal*][proposal]

## Why accept `off` as a value for `build-std`?
[rationale-build-std-off]: #why-accept-off-as-a-value-for-build-std

While not a default value, the user can specify `off` if they prefer which will
never rebuild the standard library. rustc will still return an error when the
user's target-modifiers do not match the pre-built standard library.

The `off` value is useful particularly for qualified toolchains where rebuilding
the standard library may invalidate the testing that the qualified toolchain has
undergone.

↩ [*Proposal*][proposal]

## Why add `build-std` to the `[target.<triple>]` and `[target.<cfg>]` sections?
[rationale-build-std-target-section]: #why-add-build-std-to-the-targettriple-and-targetcfg-sections

Supporting `build-std` as a key of both `[build]` and `[target]` sections allows
the greatest flexibility for the user. The overhead of rebuilding the standard
library may not be desirable in general but would be required when building on
targets which do not ship a pre-built standard library.

↩ [*Proposal*][proposal]

## Why does `[target]` take precedence over `[build]` for `build-std`?
[rationale-build-std-precedence]: #why-does-target-take-precedence-over-build-for-build-std

`[target]` configuration is necessarily more narrowly scoped so it makes sense
for it to override a global default in `[build]`.

↩ [*Proposal*][proposal]

## Why does "always" rebuild unconditionally?
[rationale-unconditional]: #why-does-always-rebuild-unconditionally

Rebuilding unconditionally avoids the complexity associated with an automatic
build-std mechanism while still being useful for users of tier three targets. By
leaving an automatic mechanism for a later stage, fewer of the technical
challenges of build-std need to be addressed all at once.

Having an opt-in mechanism initially, such as `build-std = "always"`, allows for
early issues with build-std to be ironed out without potentially affecting more
users like an automatic mechanism.

*See [Stage 2][stage2] for the introduction of an automatic build-std
mechanism.*

↩ [*Proposal*][proposal]

## Why does "always" rebuild in release profile?
[rationale-release-profile]: #why-does-always-rebuild-in-release-profile

The release profile most closely matches the existing pre-built standard
library, which has proven itself suitable for a majority of use cases.

By minimising the differences between a newly-built std and a pre-built std,
there is less chance of the user experiencing bugs or unexpected behaviour from
the well-tested and supported pre-built std.

*See [Stage 3][stage3] for the introduction of customised standard library
builds.*

↩ [*Proposal*][proposal]

## Why add `build-std-crate`?
[rationale-build-std-crate]: #why-add-build-std-crate

Not all standard library crates will build on all targets. In a `no_std` project
for a tier three target, `build-std-crate` gives the user the ability to limit
which crates are built to those they know they need and will build successfully.

*See [Stage 1b][stage1b] for an alternative to `build-std-crate`.*

↩ [*Proposal*][proposal]

## Why use the lockfile of the `rust-src` component?
[rationale-lockfile]: #why-use-the-lockfile-of-the-rust-src-component

Using different dependency versions for the standard library would invalidate
the upstream testing of the standard library. In particular, some crates use
unstable APIs when included as a dependency of the standard library meaning that
there is a high risk of build breakage if any package version is changed.

Using the lockfile included in the `rust-src` component guarantees that the same
dependency versions are used as in the pre-built standard library. As the
standard library does not re-export types from its dependencies, this will not
affect interoperability with the same dependencies of different versions used by
the user's crate.

Using the lockfile does prevent Cargo from resolving the standard library
dependencies to newer patch versions that may contain security fixes. However,
this is already impossible with the pre-built standard library.

See
[*Why vendor the standard library's dependencies?*][rationale-vendoring]

↩ [*Proposal*][proposal]

### Why not build the standard library in incremental?
[rationale-incremental]: #why-not-build-the-standard-library-in-incremental

As the standard library sources are never modified, incremental compilation
would only add a compilation time overhead.

↩ [*Proposal*][proposal]

### Why not produce a `dylib` for the standard library?
[rationale-no-dylib]: #why-not-produce-a-dylib-for-the-standard-library

The standard library supports being built as both a `rlib` and a `dylib` and
both are shipped as part of the `rust-std` component. As it does not contain a
metadata hash, it can be rebuilt unnecessarily when toolchain versions change
(e.g. switching between stable and nightly and back). The `dylib` is only linked
against when `-Cprefer-dynamic` is used. build-std will initially be
conservative and not include the `dylib`.

*See the following sections for future possibilities:*

- [*Build both `dylib` and `rlib` variants of the standard library*][future-crate-type]

↩ [*Proposal*][proposal]

## Why use the pre-built standard library for procedural macros and build-scripts?
[rationale-sysroot-for-host-deps]: #why-use-the-pre-built-standard-library-for-procedural-macros-and-build-scripts

Procedural macros and build scripts always run on the host and need to be built
with a configuration that are compatible with the host toolchain's Cargo and
rustc. There is little advantage to using a custom standard library with
procedural macros or build scripts, as they are not part of the final output
artifact and anywhere they can run already have a toolchain with host tools and
a pre-built standard library. Procedural macros must link against the compiler
which further limits potential use cases to those without `target-modifiers`.

↩ [*Proposal*][proposal]

## Why not replace `#![no_std]` as the source-of-truth for whether a crate depends on `std`?
[rationale-replace-no_std]: #why-not-replace-no_std-as-the-source-of-truth-for-whether-a-crate-depends-on-std

Crates can currently use the crate attribute `#![no_std]` to indicate a lack of
dependency on `std`. With `build-std-crates` or explicit dependencies (as in
[Stage 1b][stage1b]) allowing the user to specify a dependency on the standard
library, it is unintuitive for there to be two sources-of-truth for this
information.

`#![no_std]` serves two purposes - it stops the compiler from loading `std` from
the sysroot and adding `extern crate std`, and it prevents the user from
depending on anything from `std` accidentally.

`#![no_std]` could hypothetically be replaced by a lint to prevent use of the
standard library and a change to the compiler so that it loads the `std`
speculatively unless it is used. However, while rustc does have some support for
speculatively loading crates, it is not possible to do so and not declare them
as a dependency in cross-crate metadata.

↩ [*Interactions with `#![no_std]`*][interactions-with-no_std]

### Why remove `restricted_std`?
[rationale-remove-restricted-std]: #why-remove-restricted_std

`restricted_std` was originally added as part of a mechanism to enable the
standard library to build on all targets (just with stubbed out functionality),
however stability is not an ideal match for this use case. rustc will still try
to compile unstable code, so this won't help ensure the standard library builds
on all targets.

Furthermore, when `restricted_std` applies, users must add
`#![feature(restricted_std)]` to opt-in to using the standard library anyway
(conditionally, only for affected targets), and have no mechanism for opting-in
on behalf of their dependencies (including first-party crates like `libtest`).

It is still valuable for the standard library to be able to compile on as many
targets as possible using the `unsupported` module in its platform abstraction
layer, but this mechanism does not use `restricted_std`.

↩ [*`restricted_std`*][restricted_std]

### Why disallow custom targets?
[rationale-disallow-custom-targets]: #why-disallow-custom-targets

While custom targets can be used on stable today, in practice, they are only
used on nightly as `-Zbuild-std` would need to be used to build at least `core`.
As such, if build-std were to be stabilised, custom targets would become much
more usable on stable toolchains.

In order to avoid users relying on the [unstable target-spec-json][rust#71009]
format on a stable toolchain, using custom targets with build-std on a stable
toolchain is disallowed by Cargo until another RFC can consider all the
implications of this thoroughly. The idea of rustc disallowing custom targets on
stable is covered in [rust#71009].

↩ [*Custom targets*][custom-targets]

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

rustc's existing behaviour of implicitly loading `std` and adding it to the
extern prelude will not be changed as part of this RFC. Adding The `noprelude`
modifier for `--extern` is necessary for use of the `--extern` flag to be
equivalent to loading from a sysroot.

Without `noprelude`, rustc implicitly inserts a `extern crate $name` when using
`--extern`. As a consequence, if a newly-built `alloc` were passed using
`--extern alloc=alloc.rlib` then `extern crate alloc` would not be required to
use the locally-built `alloc`, but it would be to use the pre-built `alloc`. This
difference in how a crate is made available to rustc should not be observable to
the user.

↩ [*Preventing implicit sysroot dependencies*][preventing-implicit-sysroot-dependencies]

### Why not allow the source path for the standard library be customised?
[rationale-custom-src-path]: #why-not-allow-the-source-path-for-the-standard-library-be-customised

It is not a goal of this proposal to enable or improve the usability of custom
or modified standard libraries.

↩ [*Vendored `rust-src`*][vendored-rust-src]

### Why vendor the standard library's dependencies?
[rationale-vendoring]: #why-vendor-the-standard-librarys-dependencies

Vendoring the standard library is possible since it currently has its own
workspace, allowing the dependencies of just the standard library crates (and
not the compiler or associated tools in `rust-lang/rust`) to be easily packaged.
Doing so has multiple advantages..

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
[*Why use the lockfile of the `rust-src` component?*][rationale-lockfile]

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
crate names as, when being built as part of the standard library, dependencies
of the standard library also use unstable features and it is not practical to
special-case all of these crates.

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

### Why not globally cache builds of the standard library?
[rationale-caching]: #why-not-globally-cache-builds-of-the-standard-library

The standard library is no different than regular dependencies in being able to
benefit from global caching of dependency builds. A generic proposal for global
dependency caching could support the standard library. It is out-of-scope of
this proposal to propose a special-cased mechanism for this that applies only to
the standard library.

↩ [*Caching*][caching]

## Why not link to hosted standard library documentation in generated docs?
[rationale-generated-docs]: #why-not-link-to-hosted-standard-library-documentation-in-generated-docs

Cargo would need to pass `-Zcrate-attr="doc(html_root_url=..)"` to the standard
library crates when building them but doesn't have the required information to
know what url to provide. Cargo would require knowledge of the current toolchain
channel to build the correct url and doesn't know this.

↩ [*Generated documentation*][generated-documentation]

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The following small details are likely to be bikeshed prior to Stage 1a acceptance or
stabilisation and aren't pertinent to the overall design:

## What should the `build-std` configuration in `.cargo/config` be named?
[unresolved-config-name]: #what-should-the-build-std-configuration-in-cargoconfig-be-named

What should this configuration option be named? `build-std`?
`rebuild-standard-library`?

↩ [*Proposal*][proposal]

## What should the "always" and "off" values of `build-std` be named?
[unresolved-config-values]: #what-should-the-always-and-off-values-of-build-std-be-named

What is the most intuitive name for the values of the `build-std` setting?
`always`? `manual`? `unconditional`?

↩ [*Proposal*][proposal]

## What should `build-std-crate` be named?
[unresolved-build-std-crate-name]: #what-should-build-std-crate-be-named

What should this configuration option be named?

↩ [*Proposal*][proposal]

# Future possibilities
[future-possibilities]: #future-possibilities

There are many possible follow-ups to Stage 1a:

## Allow custom targets with build-std
[future-custom-targets]: #allow-custom-targets-with-build-std

This would require a decision from the relevant teams on the exact stability
guarantees of the target-spec-json format and whether any large changes to
the format are desirable prior to broader use.

↩ [*Custom targets*][custom-targets]

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

↩ [*Self-contained objects*][self-contained-objects]

## Build both `dylib` and `rlib` variants of the standard library
[future-crate-type]: #build-both-dylib-and-rlib-variants-of-the-standard-library

build-std could build both the `dylib` and `rlib` of the standard library.

↩ [*Why not produce a `dylib` for the standard library?*][rationale-no-dylib]

[background-dependencies]: ./1-background.md#dependencies
[future-features]: ./5-stage-1b.md#allow-enablingdisabling-features-with-build-std
[stage1b]: ./5-stage-1b.md
[stage2]: ./6-stage-2.md
[stage3]: ./7-stage-3.md

[Opaque dependencies]: https://hackmd.io/@epage/ByGfPtRell

[compiler-builtins#411]: https://github.com/rust-lang/compiler-builtins/pull/411
[compiler-team#343]: https://github.com/rust-lang/compiler-team/issues/343
[rust#76158]: https://github.com/rust-lang/rust/pull/76158
[rust#71009]: https://github.com/rust-lang/rust/pull/71009
[rust#135395]: https://github.com/rust-lang/rust/pull/135395

[std-build.rs]: https://github.com/rust-lang/rust/blob/f315e6145802e091ff9fceab6db627a4b4ec2b86/library/std/build.rs#L17

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