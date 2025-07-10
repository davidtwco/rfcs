# Appendix I: Summary of features to be implemented
[appendix-i]: #appendix-i-summary-of-features-to-be-implemented

There are many features proposed in this RFC for different parts of the project:

- Compiler
  - [`--no-implicit-sysroot-deps`][preventing-implicit-sysroot-dependencies]
  - [Loading `panic_unwind` from `-L dependency=`][rebuilding-the-standard-library]
  - [`--print target-standard-library-support`][target-standard-library-support]
- Cargo
  - [Standard library dependencies][standard-library-dependencies]
  - [Rebuilding the standard library][rebuilding-the-standard-library]
- Standard library
  - [Moving configuration into the standard library's profile][profiles]

[preventing-implicit-sysroot-dependencies]: ./4-proposal.md#preventing-implicit-sysroot-dependencies
[rebuilding-the-standard-library]: ./4-proposal.md#rebuilding-the-standard-library
[target-standard-library-support]: ./4-proposal.md#target-standard-library-support
[standard-library-dependencies]: ./4-proposal.md#standard-library-dependencies
[profiles]: ./4-proposal.md#profiles
