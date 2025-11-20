- Feature Name: `target_spec_toml`
- Start Date: 2025-11-12
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Replace rustc's existing target specification JSON format with a new extensible TOML
format.

# Motivation
[motivation]: #motivation

rustc supports built-in targets and also custom user-defined targets. There are
a handful of use-cases that custom targets address:

- Making small tweaks to existing targets to change defaults without having to
  fork the compiler
- Experimenting with new targets, or targets that cannot be upstreamed, without
  needing to fork the compiler

By definition, custom targets have no official support from the project (unlike
built-in targets, each of is supported according to their tier in the [target
tier policy]).

Custom targets are defined in a ad-hoc JSON format that has grown organically
since it was introduced. If a target isn't built-in or its name ends in `.json`
then rustc will attempt to load the target from a `$target.json` file from the
current directory or the paths from the `RUST_TARGET_PATH` environment variable.

The target-spec-json format was proposed in [rfcs#131] and was stabilised
implicitly in the merging of [rust#16156] (which added `--target` and
`RUST_TARGET_PATH`, while always loading custom targets if their name ends in
`.json` was added in [rust#49019]). rustc will permit the use of custom targets
on stable, they cannot practically be used without a build of the `core` crate,
which requires a nightly toolchain. As such, the target-spec-json format has
long been considered unstable by the compiler team, and breaking changes have
regularly been made since 2014, despite it being accepted by rustc on stable
toolchains. Formalisation of this status quo is being pursued through
destabilisation of the target-spec-json format in
[rust#71009]/[compiler-team#944] so that improvements or alternative formats can
be pursued.

[rfcs#131] did not consider a mechanism for the target-spec-json format to be
extended without new keys being immediately stabilised; whether it could scale
to supporting multiple backends; or whether JSON was the correct format compared
to TOML as is typical for the Rust toolchain (to be fair to [rfcs#131], it was
written in 2014 and lots has changed). [rfcs#131] only specified eleven keys
that would be accepted in the target-spec-json format, but it has since grown
organically [to over 120 keys][rustc-targetspecjson] without any coherent
principles guiding the growth of the format.

The typical workflow for defining a custom target is to print and modify the
target-spec-json of an existing built-in target, but as there is no mechanism
for inheriting from other targets, this results in duplication of the majority
of the target-spec-json and divergence from the built-in target as it changes.
rustc will print the target-spec-json of a built-in target with the
`--print target-spec-json` (this requires `-Zunstable-options`). Built-in
targets in rustc are defined as Rust structs and can inherit from each other.

Custom targets have previously not been subject to the same consistency checks
as built-in targets ([rust#133459]).

See also [`A-target-specs`] issues and [wg-cargo-std-aware#6] for more issues
relating to the current target-spec-json format.

[`A-target-specs`]: https://github.com/rust-lang/rust/issues?utf8=%E2%9C%93&q=sort%3Aupdated-desc+is%3Aissue+is%3Aopen+label%3AA-target-specs
[compiler-team#944]: https://github.com/rust-lang/compiler-team/issues/944
[rfcs#131]: https://rust-lang.github.io/rfcs/0131-target-specification.html
[rust#16156]: https://github.com/rust-lang/rust/pull/16156
[rust#49019]: https://github.com/rust-lang/rust/pull/49019
[rust#71009]: https://github.com/rust-lang/rust/issues/71009
[rust#133459]: https://github.com/rust-lang/rust/issues/133459
[rustc-targetspecjson]: https://github.com/rust-lang/rust/blob/d682af88a57b0045f8348507682c16c6160b522d/compiler/rustc_target/src/spec/json.rs#L480-L632
[wg-cargo-std-aware#6]: https://github.com/rust-lang/wg-cargo-std-aware/issues/6
[target tier policy]: https://doc.rust-lang.org/nightly/rustc/target-tier-policy.html

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The existing target specification format is replaced by a new TOML-based format
with an per-key notion of stability and support for target specification
inheritance.

Target specifications can be printed using the `--print target-spec` or
`--print all-target-specs` flags:

```shell-session
$ rustc --target aarch64-unknown-linux-gnu --print target-spec
...
$ rustc --print all-target-specs
```

rustc will accept target specifications as input to the `--target` flag. If a
target name ends in case-insensitive `.toml` then the rustc will attempt to load
the target specification from the filename specified in the current directory or
the directories listed in `RUST_TARGET_PATH`. If a target name does not end in a
`.toml` extension, then rustc will first attempt to match a built-in target and
fallback to loading the target from disk.

```shell-session
$ ls .
my-custom-target.toml
$ rustc --target my-custom-target.toml --print
$ rustc --target my-custom-target
```

`--print target-spec-schema` will print a [JSON Schema][json-schema] for the
TOML format proposed by this RFC.

[json-schema]: https://json-schema.org/

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This proposal does not aim to exhaustively define a `target-spec-toml` schema,
only how the format should work and the initial keys that would be included. It
is expected that the format will grow organically over time within the structure
laid out by this RFC and that unstable options will be stabilised with
appropriate consideration given to the appropriate naming and parent table when
those keys are requested by users.

Keys are either mandatory or optional. Mandatory keys must be set by a target
specification. Optional keys do not need to be set. Optional keys can have a
default value or be unset.

The values accepted by keys are subject to their own stability policies defined
on a per-key basis. For example, some keys may be passed directly to the codegen
backend and its format may change (e.g. LLVM data layouts) and so no guarantees
can be provided. Other keys, like the linker flavour, accept values entirely
chosen by the compiler team and can have stronger stability guarantees.

The target specification format has top-level keys and seven top-level tables:

- Top-level
  - Mandatory keys that describe the target being defined by the specification
  - For example, `architecture`
- `backends`
  - Keys which configure a codegen backend or indicate whether backend-specific
    features are supported by a given target
  - Each backend's table is optional and indicates whether a target supports
    that given backend
  - It is expected that if a key is relevant to multiple codegen backends then
    that the schema is shared between `backends` tables
  - For example, `backends.llvm`
- `cfgs`
  - Keys that configure cfgs that are expected for targets
  - For example, `cfgs.endian`
- `extra`
  - Table that will never be used by rustc and can be safely used by third-party
    tools
- `filenames`
  - Keys that configure the suffixes and prefixes of files that rustc outputs
    for the target (e.g. `.so`)
  - For example, `filenames.staticlib.suffix`
- `heuristics`
  - Keys that inform heuristics that influences rustc's behaviour for a target
  - For example, `heuristics.is_like_darwin`
- `linker`
  - Keys related to the rustc's invocation of the linker for this target
  - For example, `linker.supports_dynamic`
- `target`
  - Keys describing properties inherent to the target
  - For example, `target.pointer_width`

For example, consider the `aarch64-unknown-linux-gnu` target defined using the
proposed specification format:

```toml
version = "1"

architecture = "aarch64"
vendor = "unknown"
os = "linux"
environment = "gnu"

[metadata]
description = "ARM64 Linux (kernel 4.1, glibc 2.17+)"
host_tools = true
tier = 1

[cfgs]
family = "unix"

[linker]
flavor = "gnu-cc"
supports_dynamic = true
supports_thread_level = true
supports_rpath = true

[target]
pointer_width = 64
maximum_atomic_width = 128

[backends.llvm]
target = "aarch64-unknown-linux-gnu"
target_features = "+v8a,+outline-atomics"
data_layout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128-Fn32"

[unstable.backends.llvm]
default_uwtable = true
frame_pointer = "non-leaf"
stack_probes.kind = "inline"
supported_sanitizers = ["address", "leak", "memory", "thread", "hwaddress", "cfi", "memtag", "kcfi", "realtime"]
supported_split_debuginfo = ["packed", "unpacked", "off"]
supports_xray = true
target-mcount = "\u0001_mcount"
position_independent_executables = true

[unstable.linker]
crt_static_respected = true
relro_level = "full"
```

## Stability
[stability]: #stability

Target specifications are accepted by rustc on stable toolchains unless they set
keys in the `unstable` table.

New keys can be added to the target specification format by the compiler team in
the `unstable` table and this is not a breaking change. Unstable keys can be
stabilised and moved outside of the `unstable` table. The `unstable` table
serves the same purpose as the `-Z` rustc flag group.

The `unstable` table mirrors the structure of the rest of the target
specification format. For example, top-level `unstable` keys will be stabilised
as top-level keys outside of the `unstable` table and keys in
`unstable.backends.llvm` will be stabilised in the `backends.llvm` table outside
of the `unstable` table.

New keys must be optional or have a default value to avoid breaking changes to
existing target specifications accepted on stable toolchains. Every key outside
of the `extra` table is reserved and could be used by a future version of rustc.

Neither changing the default value of an existing key nor changing the value of
a key set by an built-in target are necessarily breaking changes: this depends
on the impact this would have on existing users and would be subject to the same
stability considerations as changing the default value of a compiler flag.

Keys can be deprecated and eventually ignored by rustc. Use of deprecated keys
must emit a warning informing the user that they are using a deprecated key. If
deemed appropriate, warnings should be admitted in advance of a deprecated key
having no effect. Use of deprecated keys in built-in targets will not emit a
warning. Deprecation can be used as a mechanism to rename keys.

## Inheritance
[inheritance]: #inheritance

The top-level `inherits-from` key can be used to specify another target whose
specification will be used as the baseline for the current target and whose keys
will be merged with/overridden by the current target.

`inherits-from` accepts the same values as `--target` and inherited-from targets
will be loaded according to the same rules as any other targets: matching
built-in targets first unless the target name ends in `.toml`, otherwise the
compiler will search for the target-spec in the current directory or in the
paths from `RUST_TARGET_PATH`.

Documentation for the target-spec format will include whether each key is merged
with, overrides the, or ignores the same key in the inherited-from target. For
example, consider the following keys and hypothetical behaviours:

- `metadata.description` could ignore the same key in the inherited-from target,
  and will need to be specified explicitly by the inheriting target.
- `backends.llvm.target` could override the same key in the inherited-from target.
- `backends.llvm.args` could be merged with the value of the same key in the
  inherited-from target.

Unstable keys from inherited built-in targets are respected on stable
toolchains, but cannot be overridden without the inheriting target requiring
nightly.

Diagnostics which would be emitted due to the values of or use of inherited keys
are silenced if those keys were set in a built-in target.

## Built-in targets
[built-in-targets]: #built-in-targets

rustc will define all of its built-in targets using the same target
specification format as custom targets (rustc can continue to compile these
targets into the rustc binary, rather than loading them at runtime). This will
ensure that the target specification format is fit for purpose and that all
built-in targets pass the same consistency checks required of custom targets.

## Keys
[keys]: #keys

There are [over 120 keys][rustc-targetspecjson] currently accepted by the
target-spec-json format that will need to be mapped to any new target
specification format.

Only those keys strictly necessary to define a target are being proposed as
stable keys, allowing the majority to be renamed or changed prior to their
individual stabilisation:

- Unstable keys are primarily intended to be illustrative: to give an indication
  as to how this format could be structured in practice

- Stable keys are intended to be included in the initial stabilised version of
  the format

It is a valuable exercise to demonstrate how these existing target-spec-json
keys would be mapped to the proposed format:

> [!NOTE]
>
> Many options are mapped to a key in the `backends.llvm` even if they could
> apply to multiple backends. It is expected that each backend could re-use the
> schema for keys that also apply to that backend.

- **N/A -> `version`**
  - Always set to `"1"` to enable this format to be versioned in future
  - Mandatory, does not inherit
- **`arch` -> `architecture`**
  - Architecture to use for ABI considerations and `cfg(target_arch)`
    - Combined with `env`, `os` and `vendor` to compute target tuple
  - Mandatory, does not inherit
  - Used in 306/306 built-in targets
- **`crt-objects-fallback` -> N/A**
  - Backwards compatible variant of `link-self-contained`
  - Will be replaced by `linker.link_self_contained`
  - Used in 306/306 built-in targets
- **`data-layout` -> `backends.llvm.data_layout`**
  - [Data layout][llvm-datalayout] to pass to LLVM
  - Mandatory (if LLVM backend used), overrides
  - Used in 306/306 built-in targets
- **`linker-flavor` -> `linker.flavor`**
  - Default linker flavour used if `-C linker-flavor` or `-C linker` are not
    passed on the command line
  - Optional, default depends on target-specific heuristics, overrides
  - Used in 306/306 built-in targets
- **`llvm-target` -> `backends.llvm.target`**
  - Unversioned target tuple passed to LLVM
  - Mandatory  (if LLVM backend used), overrides
  - Used in 306/306 built-in targets
- **`metadata.description` -> `metadata.description`**
  - Human-readable description of the target, used in documentation generation
  - Mandatory, does not inherit
  - Used in 306/306 built-in targets
- **`metadata.host_tools` -> `metadata.host_tools`**
  - Whether this target is distributed with host tools
  - Optional, defaults to `false`, does not inherit
  - Used in 306/306 built-in targets
- **`metadata.tier` -> `metadata.tier`**
  - Tier of the target, used in documentation generation
  - Mandatory, does not inherit
  - Used in 306/306 built-in targets
- **`metadata.std` -> N/A**
  - Target's support for the standard library
  - Will be replaced with `metadata.standard_library_support` in
    [rfcs#3874][rfcs#3784-std-support]
  - Used in 306/306 built-in targets
- **`target-pointer-width` -> `target.pointer_width`**
  - Number of bits in a pointer
    - Influences `cfg(target_pointer_width)`
  - Mandatory, overrides
  - Used in 306/306 built-in targets
- **`max-atomic-width` -> `target.maximum_atomic_width`**
  - Maximum integer size in bits that this target can perform atomic operations on
  - Optional, defaults to unset, overrides
  - Used in 298/306 built-in targets
- **`os` -> `os`**
  - Operating system name to use for `cfg(target_os)`
    - Combined with `architecture`, `env` and `vendor` to compute target tuple
  - Mandatory, does not inherit
  - Used in 255/306 built-in targets
- **`target-family` -> `cfgs.family`**
  - Values for `cfg(target_family)`
    - Common options are `"unix"` and `"windows"`
  - Optional, defaults to unset, overrides
  - Used in 227/306 built-in targets
- **`dynamic-linking` -> `linker.supports_dynamic`**
  - Whether dynamic linking is available on this target
  - Optional, defaults to `false`, overrides
  - Used in 207/306 built-in targets
- **`features` -> `backends.llvm.target_features`**
  - Default target features to pass to LLVM
    - These features overwrite `-Ctarget-cpu` but can be overwritten with
      `-Ctarget-features`
    - Corresponds to `llc -mattr=$features`
  - Optional, defaults to unset, merges
  - Used in 198/306 built-in targets
- **`cpu` -> `backends.llvm.target_cpu`**
  - Default CPU to pass to LLVM
    - Corresponds to `llc -mcpu=$cpu`
  - Optional, defaults to `"generic"`, overrides
  - Used in 176/306 built-in targets
- **`has-rpath` -> `linker.supports_rpath`**
  - Whether the linker support rpaths or not
  - Optional, defaults to `false`, overrides
  - Used in 167/306 built-in targets
- **`has-thread-local` -> `target.supports_thread_local`**
  - Whether `#[thread_local]` is available for this target
  - Optional, defaults to `false`, overrides
  - Used in 167/306 built-in targets
- **`env` -> `environment`**
  - Environment name to use for `cfg(target_env)`
    - Combined with `architecture`, `os`, and `vendor` to compute target tuple
  - Mandatory, does not inherit
  - Used in 146/306 built-in targets
- **`position-independent-executables` -> `unstable.backends.llvm.position_independent_executables`**
  - Dynamically linked executables can be compiled as position independent if
    the default relocation model of position independent code is not changed
    - This is a requirement to take advantage of ASLR, as otherwise the
      functions in the executable are not randomized and can be used during an
      exploit of a vulnerability in any code
  - Optional, defaults to `true`, overrides
  - Used in 146/306 built-in targets
- **`crt-static-respected` -> `unstable.linker.crt_static_respected`**
  - Whether or not crt-static is respected by the compiler (or is a no-op)
  - Optional, defaults to `false`, overrides
  - Used in 141/306 built-in targets
- **`linker` -> `linker.linker`**
  - Linker to invoke
  - Optional, default depends on target-specific heuristics, overrides
  - Used in 141/306 built-in targets
- **`relro-level` -> `unstable.linker.relro_level`**
  - Either partial, full, or off.
    - Full RELRO makes the dynamic linker resolve all symbols at startup and
      marks the GOT read-only before starting the program, preventing
      overwriting the GOT
  - Optional, defaults to `"off"`, overrides
  - Used in 140/306 built-in targets
- **`emit-debug-gdb-scripts` -> `unstable.backends.llvm.emit_debug_gdb_scripts`**
  - Whether a `.debug_gdb_scripts` section will be added to the output object file
  - Optional, defaults to `true`, overrides
  - Used in 118/306 built-in targets
- **`stack-probes` -> `unstable.backends.llvm.stack_probes`**
  - The implementation of stack probes to use
  - Optional, defaults to unset, overrides
  - Used in 114/306 built-in targets
- **`abi` -> `cfgs.abi`**
  - Values for `cfg(target_abi)`
    - Used to distinguish multiple ABIs on the same OS and architecture, e.g.
      `"eabi"` or `"eabihf"`
  - Optional, defaults to unset, overrides
  - Used in 113/306 built-in targets
- **`supported-split-debuginfo` -> `unstable.backends.llvm.supports_split_debuginfo`**
  - Which kinds of split debuginfo are supported by the target?
  - Optional, defaults to unset, overrides
  - Used in 113/306 built-in targets
- **`panic-strategy` -> `unstable.target.default_panic_strategy`**
  - Whether the default panic strategy for this target is `"unwind"` or
    `"abort"`
  - Optional, defaults to `"abort"`, overrides
  - Used in 107/306 built-in targets
- **`pre-link-args` -> `unstable.linker.pre_link_args`**
  - Linker arguments that are passed *before* any user-defined libraries
  - Optional, defaults to unset, merges
  - Used in 106/306 built-in targets
- **`relocation-model` -> `unstable.backends.llvm.relocation_model`**
  - Relocation model to use in object file
    - Corresponds to `llc -relocation-model=$relocation_model`
  - Optional, defaults to `"pic"`, overrides
  - Used in 89/306 built-in targets
- **`default-uwtable` -> `unstable.backends.llvm.default_uwtable`**
  - Whether or not to emit `uwtable` attributes on functions if
    `-C force-unwind-tables` is not specified and `uwtable` is not required on
    this target
  - Optional, defaults to `false`, overrides
  - Used in 88/306 built-in targets
- **`llvm-floatabi` -> `unstable.backends.llvm.floatabi`**
  - Control the float ABI to use, for architectures that support it
    - Only used on Arm
    - Corresponds to the `-float-abi` flag in llc (clang's `-mfloat-abi` is
      similar but more complicated since it can also affect the `soft-float`
      target feature)
    - If not provided, LLVM will infer the float ABI from the target triple
      (`backends.llvm.target`)
  - Optional, defaults to unset, overrides
  - Used in 88/306 built-in targets
- **`vendor` -> `vendor`**
  - Vendor name to use for `cfg(target_vendor)`
    - Combined with `architecture`, `os`, and `env` to compute target tuple
  - Mandatory, does not inherit
  - Used in 87/306 built-in targets
- **`target-mcount` -> `unstable.backends.llvm.target_mcount`**
  - Use platform-dependent mcount function
  - Optional, defaults to `"mcount"`, overrides
  - Used in 82/306 built-in targets
- **`eh-frame-header` -> `unstable.linker.eh_frame_header`**
  - Whether the linker is instructed to add a `GNU_EH_FRAME` ELF header used to
    locate unwinding information is passed (only has effect if the linker is
    `ld`-like)
  - Optional, defaults to `true`, overrides
  - Used in 73/306 built-in targets
- **`frame-pointer` -> `unstable.backends.llvm.frame_pointer`**
  - Frame pointer mode for this target
  - Optional, defaults to `"may_omit"`, overrides
  - Used in 73/306 built-in targets
- **`abi-return-struct-as-int` -> `unstable.backends.abi_return_struct_as_int`**
  - Whether the target toolchain's ABI supports returning small structs as an
    integer
  - Optional, defaults to `false`, overrides
  - Used in 64/306 built-in targets
- **`llvm-abiname` -> `unstable.backends.llvm.abi_name`**
  - Corresponds to the `-mabi` parameter available in multilib C compilers and
    the `-target-abi` flag in `llc`
  - Optional, defaults to unset, overrides
  - Used in 59/306 built-in targets
- **`linker-is-gnu` -> `unstable.linker.is_gnu`**
  - Whether the GNU linker is being used
  - Optional, defaults to `true`, overrides
  - Used in 58/306 built-in targets
- **`binary-format` -> `unstable.target.binary_format`**
  - Target's binary file format
  - Optional, defaults to `"elf"`, overrides
  - Used in 57/306 built-in targets
- **`dll-suffix` -> `filenames.dll.suffix`**
  - String to append to the name of every dynamic library
  - Optional, defaults to `".so"`, overrides
  - Used in 57/306 built-in targets
- **`supported-sanitizers` -> `unstable.backends.llvm.supported_sanitizers`**
  - The sanitizers supported by this target
    - Note that the support here is at a codegen level. If the machine code with
      sanitizer enabled can generated on this target, but the necessary
      supporting libraries are not distributed with the target, the sanitizer
      should still appear in this list for the target
  - Optional, defaults to unset, overrides
  - Used in 56/306 built-in targets
- **`exe-suffix` -> `filenames.exe.suffix`**
  - String to append to the name of every executable
  - Optional, default depends on target-specific heuristics, overrides
  - Used in 47/306 built-in targets
- **`lld-flavor` -> `unstable.linker.lld_flavor`**
  - Variant of `lld` to use
  - Optional, defaults to unset, overrides
  - Used in 46/306 built-in targets
- **`target-endian` -> `target.endianness`**
  - Values for `cfg(target_endian)`
  - Optional, defaults to little endian, overrides
  - Used in 46/306 built-in targets
- **`archive-format` -> `unstable.target.archive_format`**
  - Format that archives should be emitted in
  - Optional, defaults to unset, overrides
  - Used in 39/306 built-in targets
- **`debuginfo-kind` -> `unstable.target.debuginfo_format`**
  - Which kind of debuginfo is used by this target?
  - Optional, defaults to unset, overrides
  - Used in 38/306 built-in targets
- **`split-debuginfo` -> `unstable.backends.llvm.split_debuginfo`**
  - How to handle split debug information, if at all
  - Optional, defaults to unset, overrides
  - Used in 38/306 built-in targets
- **`default-dwarf-version` -> `unstable.backends.llvm.default_dwarf_version`**
  - Default supported version of DWARF on this platform
    - Useful because some platforms (osx, bsd) only want up to DWARF2
  - Optional, defaults to unset, overrides
  - Used in 36/306 built-in targets
- **`no-default-libraries` -> `unstable.linker.no_default_libraries`**
  - Whether to disable linking to the default libraries, typically corresponds
    to `-nodefaultlibs`
  - Optional, defaults to `true`, overrides
  - Used in 36/306 built-in targets
- **`pre-link-objects-fallback` -> `unstable.linker.pre_link_objects_fallback`**
  - Same as `pre_link_objects`, but when self-contained linking mode is enabled
  - Optional, defaults to unset, overrides
  - Used in 36/306 built-in targets
- **`plt-by-default` -> `unstable.backends.llvm.plt_by_default`**
  - Determines if the target always requires using the PLT for indirect library
    calls or not
    - Controls the default value of the `-Z plt` flag
  - Optional, defaults to `true`, overrides
  - Used in 35/306 built-in targets
- **`c-enum-min-bits` -> `unstable.target.c_enum_minimum_bits`**
  - Minimum number of bits in `#[repr(C)]` enum
  - Optional, defaults to size of `c_int`, overrides
  - Used in 33/306 built-in targets
- **`crt-static-default` -> `unstable.linker.crt_static_default`**
  - Whether or not the CRT is statically linked by default
  - Optional, defaults to `false`, overrides
  - Used in 32/306 built-in targets
- **`dll-prefix` -> `filenames.dll.prefix`**
  - String to prepend to the name of every dynamic library
  - Optional, defaults to `"lib"`, overrides
  - Used in 32/306 built-in targets
- **`tls-model` -> `unstable.backends.llvm.tls_model`**
  - TLS model to use
    - Options are `"global-dynamic"` (default), `"local-dynamic"`,
      `"initial-exec"` and `"local-exec"`
    - Similar to the `-ftls-model` option in GCC/Clang
  - Optional, defaults to `"global-dynamic"`, overrides
  - Used in 32/306 built-in targets
- **`function-sections` -> `unstable.backends.llvm.function_sections`**
  - Emit each function in its own section
  - Optional, defaults to `true`, overrides
  - Used in 31/306 built-in targets
- **`crt-static-allows-dylibs` -> `unstable.linker.crt_static_allows_dylibs`**
  - Whether or not linking dylibs to a static CRT is allowed
  - Optional, defaults to `false`, overrides
  - Used in 28/306 built-in targets
- **`code-model` -> `unstable.backends.llvm.code_model`**
  - Code model to use
    - Corresponds to `llc -code-model=$code_model`
  - Optional, defaults to unset, overrides
  - Used in 26/306 built-in targets
- **`post-link-objects-fallback` -> `unstable.linker.post_link_objects_fallback`**
  - Same as `post_link_objects`, but when self-contained linking mode is enabled
  - Optional, defaults to unset, overrides
  - Used in 26/306 built-in targets
- **`is-like-darwin` -> `heuristics.is_like_darwin`**
  - If the target toolchain is like macOS's
    - Only useful for compiling against iOS/macOS
    - Whether to run running `dsymutil` and use options like `-dead_strip`
    - Whether to use Apple-specific ABI changes, such as extending function
      parameters to 32-bits
  - Optional, defaults to `false`, overrides
  - Used in 24/306 built-in targets
- **`is-like-windows` -> `heuristics.is_like_windows`**
  - Whether the target is like Windows. This is a combination of several more
    specific properties represented as a single flag:
    - The target uses a Windows ABI, uses PE/COFF as a format for object code,
      uses Windows-style `dllexport`/`dllimport` for shared libraries, uses
      import libraries and `.def` files for symbol exports, executables support
      setting a subsystem
  - Optional, defaults to `false`, overrides
  - Used in 24/306 built-in targets
- **`link-env` -> `unstable.linker.environment`**
  - Environment variables to be set for the linker invocation
  - Optional, defaults to unset, overrides
  - Used in 24/306 built-in targets
- **`link-env-remove` -> `unstable.linker.environment_remove`**
  - Environment variables to be removed for the linker invocation
  - Optional, defaults to unset, overrides
  - Used in 24/306 built-in targets
- **`dll-tls-export` -> `unstable.linker.dll_tls_export`**
  - Whether dynamic linking can export TLS globals
  - Optional, defaults to `true`, overrides
  - Used in 23/306 built-in targets
- **`rustc-abi` -> `unstable.backends.llvm.abi`**
  - Picks a specific ABI for this target
    - This is *not* just for "Rust" ABI functions, it can also affect "C" ABI
      functions; the point is that this flag is interpreted by rustc and not
      forwarded to LLVM
    - So far, this is only used on x86
  - Optional, defaults to unset, overrides
  - Used in 22/306 built-in targets
- **`late-link-args` -> `unstable.linker.late_link_args`**
  - Linker arguments that are unconditionally passed after any user-defined but
    before post-link objects
    - Standard platform libraries that should be always be linked to, usually go
      here
  - Optional, defaults to unset, merges
  - Used in 21/306 built-in targets
- **`requires-uwtable` -> `unstable.backends.llvm.requires_uwtable`**
  - Whether or not to unconditionally `uwtable` attributes on functions,
    typically because the platform needs to unwind for things like stack
    unwinders
  - Optional, defaults to `false`, overrides
  - Used in 21/306 built-in targets
- **`static-position-independent-executables` -> `unstable.backends.llvm.static_position_independent_executables`**
  - Executables that are both statically linked and position-independent are
    supported
  - Optional, defaults to `false`, overrides
  - Used in 21/306 built-in targets
- **`supports-xray` -> `unstable.backends.llvm.supports_xray`**
  - Whether the target supports XRay instrumentation
  - Optional, defaults to `false`, overrides
  - Used in 21/306 built-in targets
- **`disable-redzone` -> `unstable.backends.llvm.disable_redzone`**
  - Do not emit code that uses the "red zone", if the ABI has one
  - Optional, defaults to `false`, overrides
  - Used in 17/306 built-in targets
- **`atomic-cas` -> `unstable.target.supports_atomic_cas`**
  - Whether the target supports atomic CAS operations natively
  - Optional, defaults to `true`, overrides
  - Used in 16/306 built-in targets
- **`singlethread` -> `unstable.target.supports_threads`**
  - Whether this target has support for threads
  - Optional, defaults to `true`, overrides
  - Used in 15/306 built-in targets
- **`is-like-msvc` -> `heuristics.is_like_msvc`**
  - Whether the target is like MSVC. This is a combination of several more
    specific properties represented as a single flag:
    - The target has all the properties from `is_like_windows`, has some MSVC-specific Windows ABI properties, uses a `link.exe`-like linker, uses CodeView/PDB for debuginfo and natvis for its visualization, uses SEH-based unwinding, supports control flow guard mechanism
    - `heuristics.is_like_msvc` implies `heurisitcs.is_like_windows`
  - Optional, defaults to `false`, overrides
  - Used in 14/306 built-in targets
- **`allows-weak-linkage` -> `unstable.linker.supports_weak_linkage`**
  - Whether the target supports weak linkage
    - The MinGW toolchain has a known issue that prevents it from correctly
      handling COFF object files with more than 2^15 sections. Since each weak
      symbol needs its own COMDAT section, weak linkage implies a large number
      sections that easily exceeds the given limit for larger codebases
  - Optional, defaults to `true`, overrides
  - Used in 13/306 built-in targets
- **`only-cdylib` -> `unstable.linker.dynamic_supports_only_cdylibs`**
  - If dynamic linking is available, whether only `cdylib`s are supported
  - Optional, defaults to `false`, overrides
  - Used in 11/306 built-in targets
- **`staticlib-prefix` -> `filenames.staticlib.prefix`**
  - String to prepend to the name of every static library
  - Optional, defaults to `"lib"`, overrides
  - Used in 11/306 built-in targets
- **`staticlib-suffix` -> `filenames.staticlib.suffix`**
  - String to append to the name of every static library
  - Optional, defaults to `".a"`, overrides
  - Used in 11/306 built-in targets
- **`use-ctors-section` -> `unstable.backends.llvm.use_ctors_section`**
  - Whether to use legacy `.ctors` initialization hooks rather than `.init_array`
  - Optional, defaults to `false`, overrides
  - Used in 11/306 built-in targets
- **`has-thumb-interworking` -> `unstable.backends.llvm.has_thumb_interworking`**
  - Whether the target is an Arm architecture using thumb v1 which allows for
    thumb and arm interworking
  - Optional, defaults to `false`, overrides
  - Used in 10/306 built-in targets
- **`generate-arange-section` -> `unstable.backends.llvm.generate_arange_section`**
  - Whether or not the DWARF `.debug_aranges` section should be generated
  - Optional, defaults to `false`, overrides
  - Used in 9/306 built-in targets
- **`is-like-wasm` -> `heuristics.is_like_wasm`**
  - Whether a target toolchain is like WASM
  - Optional, defaults to `false`, overrides
  - Used in 9/306 built-in targets
- **`main-needs-argc-argv` -> `unstable.target.main_requires_argc_argv`**
  - Whether the runtime startup code requires the `main` function be passed
    `argc` and `argv` values
  - Optional, defaults to `false`, overrides
  - Used in 9/306 built-in targets
- **`entry-name` -> `unstable.backends.llvm.entry_name`**
  - The name of entry function
  - Optional, defaults to `"main"`, overrides
  - Used in 8/306 built-in targets
- **`llvm-mcount-intrinsic` -> `unstable.backends.llvm.use_mcount_intrinsic`**
  - Use LLVM intrinsic for mcount function name
  - Optional, defaults to unset, overrides
  - Used in 8/306 built-in targets
- **`is-like-android` -> `heuristics.is_like_android`**
  - Whether a target toolchain is like Android, implying a Linux kernel and a
    Bionic libc.
  - Optional, defaults to `false`, overrides
  - Used in 7/306 built-in targets
- **`limit-rdylib-exports` -> `unstable.linker.limit_rdylib_exports`**
  - Pass a list of symbol which should be exported in the dylib to the linker
  - Optional, defaults to unset, overrides
  - Used in 7/306 built-in targets
- **`asm-args` -> `unstable.linker.asm_args`**
  - Extra arguments to pass to the external assembler (when used)
  - Optional, defaults to unset, overrides
  - Used in 5/306 built-in targets
- **`pre-link-objects` -> `unstable.linker.pre_link_objects`**
  - Objects to link before all other object code
  - Optional, defaults to unset, merges
  - Used in 5/306 built-in targets
- **`executables` -> `unstable.target.executables`**
  - Whether executables are available on this target
  - Optional, defaults to `true`, overrides
  - Used in 4/306 built-in targets
- **`is-like-solaris` -> `heuristics.is_like_solaris`**
  - Whether the target toolchain is like Solaris'
    - Only useful for compiling against Illumos/Solaris due to a different set of linker flags
  - Optional, defaults to `false`, overrides
  - Used in 4/306 built-in targets
- **`late-link-args-dynamic` -> `unstable.linker.late_link_args_dynamic`**
  - Linker arguments used in addition to `late_link_args` if at least one Rust
    dependency is dynamically linked
  - Optional, defaults to unset, merges
  - Used in 4/306 built-in targets
- **`late-link-args-static` -> `unstable.linker.late_link_args_static`**
  - Linker arguments used in addition to `late_link_args` if all Rust
    dependencies are statically linked
  - Optional, defaults to unset merges
  - Used in 4/306 built-in targets
- **`direct-access-external-data` -> `unstable.target.direct_access_external_data`**
  - Direct or use GOT indirect to reference external data symbols
  - Optional, defaults to `false`, overrides
  - Used in 3/306 built-in targets
- **`link-script` -> `unstable.linker.link_script`**
  - Optional link script applied to `dylib` and `executable` crate types
    - This is a string containing the script, not a path
    - Can only be applied to linkers where `linker.flavor == "gnu"`
  - Optional, defaults to unset, overrides
  - Used in 3/306 built-in targets
- **`llvm-args` -> `unstable.backends.llvm.args`**
  - Additional arguments to pass to LLVM
    - Similar to the `-C llvm-args` codegen option
  - Optional, defaults to unset, merges
  - Used in 3/306 built-in targets
- **`merge-functions` -> `unstable.backends.llvm.merge_functions`**
  - Determines how or whether the `MergeFunctions` LLVM pass should run for this
    target
    - Either `"disabled"`, `"trampolines"`, or `"aliases"`
    - The `MergeFunctions` pass is generally useful, but some targets may
    need to opt out
    - Optional, defaults to `"aliases"`, overrides
  - Used in 3/306 built-in targets
- **`no-builtins` -> `unstable.backends.llvm.no_builtins`**
  - Whether library functions call lowering/optimization is disabled in LLVM for
    this target unconditionally
  - Optional, defaults to `false`, overrides
  - Used in 3/306 built-in targets
- **`obj-is-bitcode` -> `unstable.linker.obj_is_bitcode`**
  - This is mainly for easy compatibility with emscripten. If we give `emcc` `.o`
    files that are actually `.bc` files it will 'just work'
  - Optional, defaults to `false`, overrides
  - Used in 3/306 built-in targets
- **`default-sanitizers` -> `unstable.backends.llvm.default_sanitizers`**
  - The sanitizers that are enabled by default on this target
    - Note that the support here is at a codegen level. If the machine code with
      sanitizer enabled can generated on this target, but the necessary
      supporting libraries are not distributed with the target, the sanitizer
      should still appear in this list for the target
  - Optional, defaults to unset, overrides
  - Used in 2/306 built-in targets
- **`is-like-gpu` -> `heuristics.is_like_gpu`**
  - Whether the target is a GPU (e.g. NVIDIA, AMD, Intel)
  - Optional, defaults to `false`, overrides
  - Used in 2/306 built-in targets
- **`min-atomic-width` -> `unstable.target.minimum_atomic_width`**
  - Minimum integer size in bits that this target can perform atomic operations on
  - Optional, defaults to unset, overrides
  - Used in 2/306 built-in targets
- **`min-global-align` -> `unstable.target.minimum_global_symbol_alignment`**
  - The minimum alignment for global symbols
  - Optional, defaults to unset, overrides
  - Used in 2/306 built-in targets
- **`need-explicit-cpu` -> `unstable.backends.need_explicit_target_cpu`**
  - Whether a cpu needs to be explicitly set
  - Optional, defaults to `false`, overrides
  - Used in 2/306 built-in targets
- **`post-link-args` -> `unstable.linker.post_link_args`**
  - Linker arguments that are passed *after* any user-defined libraries
  - Optional, defaults to unset, merges
  - Used in 2/306 built-in targets
- **`supports-stack-protector` -> `unstable.backends.llvm.supports_stack_protector`**
  - Whether the target supports stack canary checks
  - Optional, defaults to `true`, overrides
  - Used in 2/306 built-in targets
- **`target-c-int-width` -> `unstable.target.c_int_width`**
  - Width of `c_int` type
  - Optional, default to unset, overrides
  - Used in 2/306 built-in targets
- **`default-codegen-units` -> `target.default_debug_codegen_units`**
  - Default number of codegen units to use in debug mode
  - Optional, defaults to unset, overrides
  - Used in 1/306 built-in targets
- **`entry-abi` -> `unstable.backends.llvm.entry_abi`**
  -  The ABI of the entry function
  -  Optional, defaults to `"C"`, overrides
  - Used in 1/306 built-in targets
- **`is-like-aix` -> `heuristics.is_like_aix`**
  - Whether the target toolchain is like AIX
    - Linker options on AIX are special and it uses XCOFF as binary format
  - Optional, defaults to `false`, overrides
  - Used in 1/306 built-in targets
- **`is-like-vexos` -> `heuristics.is_like_vexos`**
  - Whether a target toolchain is like VEXos
  - Optional, defaults to `false`, overrides
  - Used in 1/306 built-in targets
- **`override-export-symbols` -> `unstable.linker.override_export_symbols`**
  - Have the linker export exactly these symbols, instead of using the usual
    logic to figure this out from the crate itself
  - Optional, defaults to `false`, overrides
  - Used in 1/306 built-in targets
- **`post-link-objects` -> `unstable.linker.post_link_objects`**
  - Objects to link after all other object code
  - Optional, defaults to unset, merges
  - Used in 1/306 built-in targets
- **`relax-elf-relocations` -> `unstable.linker.relax_elf_relocations`**
  - Whether or not RelaxElfRelocation flag will be passed to the linker
  - Optional, defaults to `false`, overrides
  - Used in 1/306 built-in targets
- **`requires-lto` -> `unstable.target.requires_lto`**
  - This target requires everything to be compiled with LTO to emit a final
    executable (i.e. there is no native linker for this target)
  - Optional, defaults to `false`, overrides
  - Used in 1/306 built-in targets
- **`simd-types-indirect` -> `unstable.backends.llvm.simd_types_indirect`**
  - Whether or not SIMD types are passed by reference in the Rust ABI
    - Typically required if a target can be compiled with a mixed set of target
      features. This is `true` by default, and `false` for targets like wasm32
      where the whole program either has simd or not
  - Optional, default depends on target-specific heuristics, overrides
  - Used in 1/306 built-in targets
- **`trap-unreachable` -> `unstable.backends.llvm.trap_unreachable`**
  - Whether to generate trap instructions in places where optimization would
    otherwise produce control flow that falls through into unrelated memory
  - Optional, defaults to `false`, overrides
  - Used in 1/306 built-in targets
- **`link-self-contained` -> `unstable.linker.link_self_contained`**
  - Behaviour for the self-contained linking mode: inferred for some targets, or
    explicitly enabled (in bulk, or with individual components)
  - Optional, defaults to unset, overrides
  - Used in 0/306 built-in targets
- **`allow-asm` -> `unstable.target.supports_asm`**
  - Whether `asm!()` is supported
  - Optional, defaults to `true`, overrides
  - Used in 0/306 built-in targets
- **`default-codegen-backend` -> `default_backend`**
  - Default codegen backend used for this target
    - Overridden by `CFG_DEFAULT_CODEGEN_BACKEND` environmental variable
      captured when compiling `rustc` will be used instead (or `"llvm"` if
      unset)
    - Can always be overridden with `-Zcodegen-backend`
  - Mandatory if options for multiple backends are provided, defaults to unset,
    overrides
  - Used in 0/306 built-in targets
- **`default-visibility` -> `unstable.target.default_symbol_visibility`**
  - The default visibility for symbols in this target
  - Optional, defaults to unset, overrides
  - Used in 0/306 built-in targets
- **`small-data-threshold-support` -> `unstable.backends.llvm.supports_small_data_threshold`**
  - Whether the targets supports `-Z small-data-threshold`: controls the maximum
    static variable size that may be included in the "small data sections"
    (`.sdata`, `.sbss`) supported by some architectures (RISCV, MIPS, M68K,
    Hexagon)
  - Optional, defaults to `false`, overrides
  - Used in 0/306 built-in targets
- **`default-address-space` -> `unstable.backends.llvm.default_address_space`**
  - The default address space for this target
    - Most targets simply use LLVM's default address space (`0`). Some other
      targets, such as CHERI targets, use a custom default address space (in
      this specific case, `200`)
  - Optional, defaults to `0`, overrides
  - This field exists in the `rustc_target::spec::Target` type but not in the
    JSON format

The following command line invocation can be used to see which targets use any
of the keys above where that has not been included:

```shell-session
$ rustc +nightly --print all-target-specs-json -Zunstable-options | jq -r --arg key "trap-unreachable" 'to_entries | map(select(.value[$key] != null) | .key)'
[
  "msp430-none-elf"
]
```

[llvm-datalayout]: https://llvm.org/docs/LangRef.html#data-layout
[rfcs#3784-std-support]: https://github.com/davidtwco/rfcs/blob/build-std-part-two-always/text/3874-build-std-always.md#standard-library-crate-stability

# Drawbacks
[drawbacks]: #drawbacks

Existing target specifications are used on nightly by many different users and
this would result in lots of churn for those users.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Revisiting the target specification format is necessary as there is a consensus
within the compiler team that the current format is not desirable. Current
efforts to advance build-std would make it easier to build `core` for custom
targets and increase their usability on stable toolchains, making it harder to
make changes - as such, some alternative or improvements to the current target
specification format are necessary in the short-term.

## Why use TOML over JSON?
[rationale-why-use-toml-over-json]: #why-use-toml-over-json

TOML is the format used by the rest of the Rust toolchain and is better-suited
to hand-written inputs, like these target specifications. There are no
advantages that JSON has over TOML that are compelling enough for this use case
to justify its choice over the "default" for Rust of TOML.

## Why introduce notions of stability to the target specification format?
[rationale-why-stability-in-the-format]: #why-introduce-notions-of-stability-to-the-target-specification-format

Per-key stability is crucial to enabling this format to be extended over time
and allow experimentation with the representation of different options within
the target specification format prior to committing to indefinite support. The
process of stabilising individual keys will be familiar to the compiler team and
has many parallels in compilation flags and compiler features.

# Prior art
[prior-art]: #prior-art

rustc's existing JSON target specification format was defined in [rfcs#131] -
describing the format of a target tuple, the initial twelve keys of the JSON
format, how the target specifications are loaded and how the keys would be used.

The author is not aware of any other programming languages that permit
user-defined targets.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

There are many unresolved questions related to which keys should be present in a
target specification format, as well as related to the naming and organisation
of those keys. By introducing the notion of per-key stability, this RFC aims to
delay consideration of these questions until the later stabilisation of
individual keys, and instead gain consensus on the general structure of this format.

## Which keys should be stable initially?
[unresolved-initial-stable]: #which-keys-should-be-stable-initially

It is unclear which keys are necessary to include in an initial MVP of the
target specification format so that it could be useful for some existing users
of the target specification format.

# Future possibilities
[future-possibilities]: #future-possibilities

There are a handful of future possibilities that this could enable:

## Generating target documentation from target specifications
[future-docs]: #generating-target-documentation-from-target-specifications

Documentation for targets could be added to the target specification format and
used to generate rustc's platform support documentation.

## Using target specifications from other tools
[future-other-tools]: #using-target-specifications-from-other-tools

leveraged by different parts of the toolchain (e.g. Cargo could also depend on
rustc's library for parsing and understanding target specifications could be
this library to be able to understand targets better - e.g. if target aliases
were later introduced). Third-party tools could add their own metadata to the
format under the `extra` key.
