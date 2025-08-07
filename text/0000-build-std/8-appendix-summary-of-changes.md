# Appendix I: Summary of changes
[appendix-i]: #appendix-i-summary-of-changes

There are many features proposed in this RFC for different parts of the project:

- Bootstrap/infra/release
  - [Vendoring standard library sources into `rust-src`](./4-stage-1a.md#vendored-rust-src) (Stage 1a)
  - [`rust-src` is a default component](./4-stage-1a.md#vendored-rust-src) (Stage 1a)
  - [`rust-self-contained` components](./4-stage-1a.md#special-object-files) (Stage 1a)
  - [Testing build-std in rust-lang/rust CI][constraints-on-the-standard-library]
- Cargo
  - [`build-std = "always"`](./4-stage-1a.md) (Stage 1a)
    - [Extending Cargo subcommmands](./4-stage-1a.md#cargo-subcommands)
  - [Prohibiting custom targets](./4-stage-1a.md#custom-targets) (Stage 1a)
  - [Explicit dependencies](./5-stage-1b.md#proposal) (Stage 1b)
    - [Lockfile changes](./5-stage-1b.md#proposal)
    - [Registry changes](./5-stage-1b.md#registries)
    - [Extending Cargo subcommmands](./5-stage-1b.md#cargo-subcommands)
  - [`build-std = "compatible"`](./6-stage-2.md#proposal) (Stage 2)
  - [`build-std = "compatible-profile"` / `build-std = "match-profile"`](./7-stage-3.md) (Stage 3)
- Compiler
  - [Loading `panic_unwind` from `-L dependency=`](./4-stage-1a.md#proposal) (Stage 1a)
  - [`--no-implicit-sysroot-deps`](./4-stage-1a.md#preventing-implicit-sysroot-dependencies) (Stage 1a)
  - [Assuming `RUSTC_BOOTSTRAP` for sysroot builds](./4-stage-1a.md#building-the-standard-library-on-a-stable-toolchain) (Stage 1a)
  - [Forcing many codegen-units for `compiler-builtins`](./4-stage-1a.md#compiler-builtins) (Stage 1a)
  - [Checking compatibility of flags and rlibs](./6-stage-2.md#proposal) (Stage 2)
- Project-wide
  - [Documenting build-std stability guarantees](./4-stage-1a.md#stability-guarantees) (Stage 1a)
- Standard library
  - [Removing `restricted_std`](./4-stage-1a.md#restricted_std) (Stage 1a)
  - [Moving configuration into the standard library's profile](./4-stage-1a.md) (Stage 1a)

## Constraints on the standard library, compiler and bootstrap
[constraints-on-the-standard-library]: #constraints-on-the-standard-library-compiler-and-bootstrap

A stable mechanism for building the standard library imposes some constraints on
the rest of the toolchain that would need to be upheld:

- No further customisation of the pre-built standard library through any means
  other than the profile in `Cargo.toml`
- No new C dependencies on the standard library
- The standard library continues to exist in its own workspace, with its own
  lockfile
- The name of the `test` crate becomes stable (but not its interface)
- The `panic-unwind` and `compiler-builtins-mem` `sysroot` features become
  stable so Cargo can refer to them
  - This should not necessitate a "stable/unstable features" mechanism, rather a
    guarantee from the library team that they're happy for these to stay

> [!NOTE]
>
> Cargo could be made a [JOSH] subtree of the [rust-lang/rust] so that all
> relevant parts of the toolchain can be updated in tandem when this is
> necessary.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

The following are all of the unresolved questions from all stages of the RFC:

- [*What should the `build-std` configuration in `.cargo/config` be named?*][unresolved-config-name]
- [*What should the "always" and "never" values of `build-std` be named?*][unresolved-config-values]
- [*What should `build-std-crate` be named?*][unresolved-build-std-crate-name]

## Future possibilities
[future-possibilities]: #future-possibilities

The following are all of the future possibilities from all stages of the RFC:

- [*Allow custom targets with build-std*][future-custom-targets]
- [*Avoid building `panic_unwind` unnecessarily*][future-panic_unwind]
- [*Enable local recompilation of special object files/sanitizer runtimes*][future-recompile-special]
- [*Allow choosing the crate type of the standard library?*][future-crate-type]

[future-crate-type]: ./4-stage-1a.md#allow-choosing-the-crate-type-of-the-standard-library
[future-custom-targets]: ./4-stage-1a.md#allow-custom-targets-with-build-std
[future-panic_unwind]: ./4-stage-1a.md#avoid-building-panic_unwind-unnecessarily
[future-recompile-special]: ./4-stage-1a.md#enable-local-recompilation-of-special-object-filessanitizer-runtimes
[unresolved-build-std-crate-name]: ./4-stage-1a.md#what-should-build-std-crate-be-named
[unresolved-config-name]: ./4-stage-1a.md#what-should-the-build-std-configuration-in-cargoconfig-be-named
[unresolved-config-values]: ./4-stage-1a.md#what-should-the-always-and-never-values-of-build-std-be-named

[JOSH]: https://josh-project.github.io/josh/intro.html
[rust-lang/rust]: https://github.com/rust-lang/rust