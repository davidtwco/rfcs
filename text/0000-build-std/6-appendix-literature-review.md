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
[a1-rfcs-1133-2015]: #rfcs1133-2015

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
[a1-xargo-and-cargo-4959-2016]: #xargo-and-cargo4959-2016

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
[a1-rfcs-2663-2019]: #rfcs2663-2019

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
[a1-wg-cargo-std-aware-2019-]: #wg-cargo-std-aware-2019-

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

[history]: ./2-history.md

[JOSH]: https://josh-project.github.io/josh/intro.html
[cargo-xbuild]: https://github.com/rust-osdev/cargo-xbuild
[embedded-wg]: https://github.com/rust-embedded/wg
[portability-wg]: https://github.com/rust-lang-nursery/portability-wg
[rust-lang/cargo]: https://github.com/rust-lang/cargo
[rust-lang/rust]: https://github.com/rust-lang/rust
[wg-cargo-std-aware]: https://github.com/rust-lang/wg-cargo-std-aware
[xargo]: https://github.com/japaric/xargo

[A vision for portability in Rust]: http://aturon.github.io/tech/2018/02/06/portability-vision/

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
[internals.r-l.o: Fleshing out libstd scenarios]: https://internals.rust-lang.org/t/fleshing-out-libstd-scenarios/4206
[internals.r-l.o: Refactoring libstd for ultimate portability]: https://internals.rust-lang.org/t/refactoring-std-for-ultimate-portability/4301
[jamesmunns/rfcs#1]: https://github.com/jamesmunns/rfcs/pull/1
[rfcs#1133]: https://github.com/rust-lang/rfcs/pull/1133
[rfcs#1502]: https://github.com/rust-lang/rfcs/pull/1502
[rfcs#1868]: https://github.com/rust-lang/rfcs/pull/1868
[rfcs#2663-t-lang]: https://github.com/rust-lang/lang-team/blob/master/minutes/2019-06-06.md?rgh-link-date=2019-06-06T23%3A20%3A17Z-
[rfcs#2663]: https://github.com/rust-lang/rfcs/pull/2663
[rust#107016]: https://github.com/rust-lang/rust/issues/107016
[rust#108924]: https://github.com/rust-lang/rust/pull/108924
[rust#119899]: https://github.com/rust-lang/rust/pull/119899
[rust#120232]: https://github.com/rust-lang/rust/pull/120232
[rust#123360]: https://github.com/rust-lang/rust/pull/123360
[rust#128534]: https://github.com/rust-lang/rust/pull/128534
[rust#135395]: https://github.com/rust-lang/rust/pull/135395
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
[rust#77086]: https://github.com/rust-lang/rust/pull/77086
[rust#78790]: https://github.com/rust-lang/rust/pull/78790
[rust#79218]: https://github.com/rust-lang/rust/pull/79218
[rust#79838]: https://github.com/rust-lang/rust/pull/79838
[rust#79958]: https://github.com/rust-lang/rust/pull/79958
[rust#80082]: https://github.com/rust-lang/rust/pull/83655
[rust#83655]: https://github.com/rust-lang/rust/pull/83655
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

[embed-rs-cargo-toml]: https://github.com/embed-rs/stm32f7-discovery/blob/e2bf713263791c028c2a897f2eb1830d7f09eceb/Cargo.toml#L21
[embed-rs-source]: https://github.com/embed-rs/stm32f7-discovery/blob/e2bf713263791c028c2a897f2eb1830d7f09eceb/core/src/lib.rs#L7
[std-build.rs]: https://github.com/rust-lang/rust/blob/f315e6145802e091ff9fceab6db627a4b4ec2b86/library/std/build.rs#L17

[12101111]: https://github.com/12101111
[AZMCode]: https://github.com/AZMCode
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
[jschwe]: https://github.com/jschwe
[jyn514]: https://github.com/jyn514
[ketsuban]: https://github.com/ketsuban
[madsmtm]: https://github.com/madsmtm
[mkb2091]: https://github.com/mkb2091
[nagisa]: https://github.com/nagisa
[nazar-pc]: https://github.com/nazar-pc
[parraman]: https://github.com/parraman
[phip1611]: https://github.com/phip1611
[raoulstrackx]: https://github.com/raoulstrackx
[raphaelcohn]: https://github.com/raphaelcohn
[rust-osdev]: https://github.com/rust-osdev
[saethlin]: https://github.com/saethlin
[skyzh]: https://github.com/skyzh
[tamird]: https://github.com/tamird
[tmiasko]: https://github.com/tmiasko
[tomaak]: https://github.com/tomaak
[vi]: https://github.com/vi
[wcampbell0x2a]: https://github.com/wcampbell0x2a
[weihanglo]: https://github.com/weihanglo
[yogh333]: https://github.com/yogh333
