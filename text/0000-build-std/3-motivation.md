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
compelling use-case is presented, and so this RFC may make decisions which make
these motivations harder to solve in future:

1. **Modifying the source code of the standard library** ([wg-cargo-std-aware#7])

  - Some platforms require a heavily modified standard library that would not
    be suitable for upstreaming, such as [Apache's SGX SDK][sgx] which replaces
    some standard library and ecosystem crates with forks or custom crates for a
    custom `x86_64-unknown-linux-sgx` target
  - Similarly, some tier three targets may wish to patch standard library
    dependencies to add or improve support for the target

[cargo-xbuild]: https://github.com/rust-osdev/cargo-xbuild
[xargo]: https://github.com/japaric/xargo

[rfcs#3716]: https://rust-lang.github.io/rfcs/3716-target-modifiers.html
[rust#136966]: https://github.com/rust-lang/rust/issues/136966
[wg-cargo-std-aware#2]: https://github.com/rust-lang/wg-cargo-std-aware/issues/2
[wg-cargo-std-aware#4]: https://github.com/rust-lang/wg-cargo-std-aware/issues/4
[wg-cargo-std-aware#7]: https://github.com/rust-lang/wg-cargo-std-aware/issues/7

[sgx]: https://github.com/apache/incubator-teaclave-sgx-sdk