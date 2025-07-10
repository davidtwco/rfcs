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
named `core`, `alloc` or `std` ([?][rationale-no-builtin-other-crates]) on
stable, but can be specified freely on nightly
([?][rationale-nightly-builtin-crates]]).

Crates without an explicit dependency on the standard library now have a
implicit dependency ([?][rationale-no-migration]) on `std`, `alloc` and `core`
crates ([?][rationale-implicit-direct-deps]). In the `hello_world` crate below,
there are no explicit `builtin` dependencies..

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
```

..which is equivalent to the following explicit dependencies:

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
dependency on `std`.

crates.io will accept crates published which have `builtin` dependencies.

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
([?][rationale-package-key]). It is not possible to perform source replacement
on the `builtin` source using the `[source]` Cargo config table
([?][rationale-source-replacement], [future-possibilities]).

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
- [*Why imply a direct dependency on all of `std`, `alloc` and `core`?*][rationale-implicit-direct-deps]
- [*Why disallow builtin dependencies on other crates?*][rationale-no-builtin-other-crates]
- [*Why allow all names for `builtin` crates on nightly?*][rationale-nightly-builtin-crates]
- [*Why not migrate to always requiring explicit standard library dependencies?*][rationale-no-migration]
- [*Why disallow renaming standard library dependencies?*][rationale-package-key]
- [*Why disallow source replacement on `builtin` packages?*][rationale-source-replacement]
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

In line with `crates.io`'s policy of not allowing packages with dependencies on
code published outside of `crates.io`, crates with these dependency sources will
not be able to be published to `crates.io`.

*See the following sections for rationale/alternatives:*

- [*Why permit patching of the standard library dependencies on nightly?*][rationale-patching]

*See the following sections for relevant unresolved questions:*

- [*What syntax is used to patch dependencies on the standard library in `Cargo.toml`?*][unresolved-patch-syntax]

### Features
[features]: #features

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

- [*Why limit enabling standard library features to nightly?*][rationale-features]

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
library's workspace to point to the local copy of the crates. This avoids
`crates.io` dependencies needing to add support for `rustc_dep_of_std` before
the standard library can depend on them.

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
[registries]: #registries

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
"off" ([?][rationale-build-std-off]), "target-modifiers" or "match-profile":

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
> standard library is resolved differently to other dependencies:
>
> - The lockfile included in the standard library source will be used when
>   resolving the standard library's dependencies ([?][rationale-lockfile]).
> - The dependencies of the standard library crates are entirely opaque to the
>   user. A different semver-compatible version of standard library
>   dependencies can exist in the user's resolve, and the user cannot control
>   compilation any of the dependencies of the `core`, `alloc`  or `std`
>   standard library crates individually.
> - The profile defined in the standard library will be used (see [profiles]).
>
> Cargo will resolve an opaque dependency like the standard library separately,
> and will load its workspace and perform that part of the resolve in it. The
> roots for the resolve consist of the unified set of packages that any crate in
> the dependency graph has a explicit dependency on and those which Cargo infers
> a direct dependency on, including `test` when appropriate. The resolver will
> add relevant dependencies on these root crates for crates in the "parent"
> resolve.
>
> rustc loads panic runtimes in a different way to most dependencies, and
> without looking in the sysroot they will fail to load correctly unless passed
> in with `--extern`. rustc will need to be patched to be able to load panic
> runtimes from `-L dependency=` paths in line with other transitive
> dependencies.
>
> The standard library will always be a non-incremental build
> ([?][rationale-incremental]), with no `depinfo` produced, and only a `rlib`
> produced (no `dylib`) ([?][rationale-no-dylib]). It will be built into the
> `target` directory of the crate or workspace like any other dependency.

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
[panic-strategies]: #panic-strategies

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

When building an existing `no_std` project for target without `std` support,
there could be an implicit dependency on `std` from a dependency `no_std` crate
that has not yet made its `builtin` dependencies explicit. In this circumstance
with `build-std` enabled this would fail to build as the target will have
`standard_library_support.std = false` in its target specification and Cargo
will refuse to build `std` (see
[*Target standard library support*][target-standard-library-support]).
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

### Rust Project support
[rust-project-support]: #rust-project-support

As per [background-target-support], the Rust project maintains CI jobs that run
tests for tier 1 targets. It is infeasible for these CI jobs to cover the wide
variety of user configurations for the standard library that `build-std` allow
for. For this reason, for the purposes of support and bug prioritisation, a
target rebuilding the standard library from source should be treated as no
higher than tier 2.

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

### Why allow all names for `builtin` crates on nightly?
[rationale-nightly-builtin-crates]: #why-allow-all-names-for-builtin-crates-on-nightly

Given that all standard library crates valid for that target are currently
available in the sysroot, the user can write an `extern crate` declaration and
make them available in their crate. All crates other than `std`, `alloc` or
`core` are marked unstable either explicitly or implicitly with the use of
`-Zforce-unstable-if-unmarked` meaning that these `extern crate` declarations
require opting into the crates instability.

An example is that many users have written benchmarks using `test` and have
written `extern crate test` not gated on `#[cfg(test)]` attribute. These users
need a way to specify their `test` dependency. There may be other niche uses of
unstable sysroot crates that would be unable to work correctly with this RFC.

All names are permitted for `builtin` crates rather than an allowlist to avoid
Cargo needing to hardcode the names of many of the crates in the sysroot, which
are inherently unstable.

↩ [*Standard library dependencies*][standard-library-dependencies]

### Why imply a direct dependency on all of `std`, `alloc` and `core`?
[rationale-implicit-direct-deps]: #why-imply-a-direct-dependency-on-all-of-std-alloc-and-core

When a crate depends on `std`, the user can also write `extern crate alloc` or
similar for `core`. From Cargo's perspective this adds a direct dependency on
these crates, which should always be present for compatibility purposes.

Cargo passes direct dependencies of the current crate with the `--extern` flag
and passes the `-L dependency=...` flag so rustc can search for transitive
dependencies itself. Looking for direct dependencies in a `-L crate=...`
directory would create the possibility of rustc finding stale artifacts from
previous builds. As a consequence, Cargo must be aware of the names of any
direct dependencies of a crate and cannot rely on the fact that they are part of
the dependency graph below the crate.

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

### Why disallow source replacement on `builtin` packages?
[rationale-source-replacement]: #why-disallow-source-replacement-on-builtin-packages

As [previously stated][vendored-rust-src] modifying the source code of the
standard library in the `rust-src` component is not permitted. Source
replacement of the `builtin` source could be a way to support this in the future
but it is not clear at this time what the exact use cases for doing this are and
whether the Rust Project wishes to support this. For these reasons it is left as
a possible future extension to this RFC.

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
`core`, third party registries may allow them without a warning. The schema
needs a way to refer to packages with the same name either in the registry or
builtin, which `builtin_deps` allows.

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
[*Why not ship a debug profile `rust-std?*][rationale-ship-debug-std]
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
library works as expected for a target, per the
[target tier policy][target-tier-policy]. In particular, some crates use
unstable APIs when included as a dependency of the standard library meaning that
there is a high risk of build breakage if any package version is changed.

Using the lockfile included in the `rust-src` component guarantees that the same
dependency versions are used as in the pre-built standard library. As the
standard library `crates.io` dependencies are private, it does not re-export
types from its dependencies, this will not affect interoperability with the
same dependencies of different versions used by the user's crate.

Using the lockfile does prevent Cargo from resolving the standard library
dependencies to newer patch versions that may contain security fixes. However,
this is already impossible with the prebuilt standard library.

See
[*Why vendor the standard library's dependencies?*][rationale-vendoring]

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

## Allow `builtin` source replacement
[future-source-replacement]: #allow-builtin-source-replacement

This involves allowing the user to blanket-override the standard library sources
with a `[source.builtin]` section of the Cargo configuration.

As [rationale-source-replacement] details it is unclear if users need to do this
or if it's even something the Rust project wishes to support.

↩ [*Standard library dependencies*][standard-library-dependencies]

[background-target-support]: ./1-background.md#target-support
[rfcs-2663-2019]: ./2-history.md#rfcs2663-2019
[xargo-and-cargo-4959-2016]: ./2-history.md#xargo-and-cargo4959-2016
[motivation]: ./3-motivation.md

[JOSH]: https://josh-project.github.io/josh/intro.html
[rust-lang/rust]: https://github.com/rust-lang/rust

[Opaque dependencies]: https://hackmd.io/@epage/ByGfPtRell

[compiler-builtins#411]: https://github.com/rust-lang/compiler-builtins/pull/411
[compiler-team#343]: https://github.com/rust-lang/compiler-team/issues/343
[rust#123617]: https://github.com/rust-lang/rust/pull/123617
[rust#71009]: https://github.com/rust-lang/rust/pull/71009
[rust#76185]: https://github.com/rust-lang/rust/pull/76185

[cargo-docs-renaming]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#renaming-dependencies-in-cargotoml
[cargo-json-schema]: https://doc.rust-lang.org/cargo/reference/registry-index.html#json-schema
[embed-rs-source]: https://github.com/embed-rs/stm32f7-discovery/blob/e2bf713263791c028c2a897f2eb1830d7f09eceb/core/src/lib.rs#L7
[rust-extern-prelude]: https://doc.rust-lang.org/reference/names/preludes.html#extern-prelude
[target-tier-policy]: https://doc.rust-lang.org/nightly/rustc/target-tier-policy.html
[std-build.rs]: https://github.com/rust-lang/rust/blob/f315e6145802e091ff9fceab6db627a4b4ec2b86/library/std/build.rs#L17
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