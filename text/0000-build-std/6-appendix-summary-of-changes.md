# Appendix I: Summary of changes
[appendix-i]: #appendix-i-summary-of-changes

There are many features proposed in this RFC for different parts of the project:

- Bootstrap/infra/release
  - [Vendoring standard library sources into `rust-src`](./4-build-std-always.md#vendored-rust-src) (`build-std=always`)
  - [`rust-src` is a default component](./4-build-std-always.md#vendored-rust-src) (`build-std=always`)
  - [`rust-self-contained` components](./4-build-std-always.md#self-contained-objects) (`build-std=always`)
  - [Testing build-std in rust-lang/rust CI][constraints-on-the-standard-library]
- Cargo
  - [`build-std = "always"`](./4-build-std-always.md) (`build-std=always`)
    - [Extending Cargo subcommmands](./4-build-std-always.md#cargo-subcommands)
  - [Prohibiting custom targets](./4-build-std-always.md#custom-targets) (`build-std=always`)
  - [Explicit dependencies](./5-standard-library-dependencies.md#proposal) (*Standard library dependencies*)
    - [Lockfile changes](./5-standard-library-dependencies.md#proposal)
    - [Registry changes](./5-standard-library-dependencies.md#registries)
    - [Extending Cargo subcommmands](./5-standard-library-dependencies.md#cargo-subcommands)
- Compiler
  - [Loading `panic_unwind` from `-L dependency=`](./4-build-std-always.md#proposal) (`build-std=always`)
  - [`--no-implicit-sysroot-deps`](./4-build-std-always.md#preventing-implicit-sysroot-dependencies) (`build-std=always`)
  - [Destabilise custom targets](./4-build-std-always.md#custom-targets) (`build-std=always`)
  - [Assuming `RUSTC_BOOTSTRAP` for sysroot builds](./4-build-std-always.md#building-the-standard-library-on-a-stable-toolchain) (`build-std=always`)
  - [Detect missing `rust-self-contained` components and provide diagnostics](./4-build-std-always.md#self-contained-objects) (`build-std=always`)
  - [Forcing many codegen-units for `compiler-builtins`](./4-build-std-always.md#compiler-builtins) (`build-std=always`)
- Project-wide
  - [Documenting build-std stability guarantees](./4-build-std-always.md#stability-guarantees) (`build-std=always`)
- Standard library
  - [Removing `restricted_std`](./4-build-std-always.md#restricted_std) (`build-std=always`)
  - [Moving configuration into the standard library's profile](./4-build-std-always.md) (`build-std=always`)

## Constraints on the standard library, compiler and bootstrap
[constraints-on-the-standard-library]: #constraints-on-the-standard-library-compiler-and-bootstrap

A stable mechanism for building the standard library imposes some constraints on
the rest of the toolchain that would need to be upheld:

- No further required customisation of the pre-built standard library through
  any means other than the profile in `Cargo.toml`
- Avoid mandatory C dependencies on the standard library
  - At the very least, new dependencies on the standard library will impact
    whether the standard library can be successfully built by users with varying
    environments and this impact will need to be considered going forward
  - New C dependencies will need to be careful not to cause symbol conflicts
    with user crates that pull in the same dependency (e.g. using
    [`links =...`][links])
    - If this did come up, it might be possible to work around it with
      postprocessing that renames C symbols used by the standard library but
      that would be better avoided
- The standard library continues to exist in its own workspace, with its own
  lockfile
- The name of the `test` crate becomes stable (but not its interface)
- The `panic-unwind` and `compiler-builtins-mem` `sysroot` features become
  stable so Cargo can refer to them
  - This should not necessitate a "stable/unstable features" mechanism, rather a
    guarantee from the library team that they're happy for these to stay
- Dependencies of the standard library cannot use build probes to detect whether nightly features can be used
  - With
    [*Assuming `RUSTC_BOOTSTRAP` for sysroot builds*](./4-build-std-always.md#building-the-standard-library-on-a-stable-toolchain),
    these build probes would always assume the crate is being built on nightly

> [!NOTE]
>
> Cargo will likely be made a [JOSH] subtree of the [rust-lang/rust] so that all
> relevant parts of the toolchain can be updated in tandem when this is
> necessary.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

The following are all of the unresolved questions from all stages of the RFC:

- [*What should the `build-std` configuration in `.cargo/config` be named?*][unresolved-config-name]
- [*What should the "always" and "never" values of `build-std` be named?*][unresolved-config-values]
- [*What should `build-std-crates` be named?*][unresolved-build-std-crate-name]

## Future possibilities
[future-possibilities]: #future-possibilities

The following are all of the future possibilities from all stages of the RFC:

- [*Allow custom targets with build-std*][future-custom-targets]
- [*Avoid building `panic_unwind` unnecessarily*][future-panic_unwind]
- [*Enable local recompilation of special object files/sanitizer runtimes*][future-recompile-special]
- [*Allow choosing the crate type of the standard library?*][future-crate-type]

[future-crate-type]: ./4-build-std-always.md#build-both-dylib-and-rlib-variants-of-the-standard-library
[future-custom-targets]: ./4-build-std-always.md#allow-custom-targets-with-build-std
[future-panic_unwind]: ./4-build-std-always.md#avoid-building-panic_unwind-unnecessarily
[future-recompile-special]: ./4-build-std-always.md#enable-local-recompilation-of-special-object-filessanitizer-runtimes
[unresolved-build-std-crate-name]: ./4-build-std-always.md#what-should-build-std-crates-be-named
[unresolved-config-name]: ./4-build-std-always.md#what-should-the-build-std-configuration-in-cargoconfig-be-named
[unresolved-config-values]: ./4-build-std-always.md#what-should-the-always-and-never-values-of-build-std-be-named

[JOSH]: https://josh-project.github.io/josh/intro.html
[rust-lang/rust]: https://github.com/rust-lang/rust
[links]: https://doc.rust-lang.org/nightly/cargo/reference/manifest.html#the-links-field