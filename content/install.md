+++
title = "Install"
template = "page.html"
+++

Quarz has no release yet: build it from source.

## Requirements

- [Rust](https://www.rust-lang.org/tools/install) 1.96 or later, with Cargo.
- clang 14 or later, for `quarz build` (compiled programs go through
  LLVM IR). `quarz run` does not need it.
- Linux on x86-64: CI runs there only, and other systems are untested.

## Build

```sh
git clone https://github.com/quarz-lang/quarz.git
cd quarz
cargo build --release
```

The command is `target/release/quarz`; put that directory on your `PATH`
or copy the file to one that is.

## Run a program

```sh
quarz run examples/hello.qz        # type-check and run
quarz check examples/shapes.qz     # type-check only
quarz build examples/shapes.qz -o shapes && ./shapes   # compile to a native executable
quarz repl                         # interactive session
```

A program runs the same with `quarz run` and compiled with `quarz build`;
the interpreter starts at once, the executable runs faster. `quarz test`
runs a cog's tests, and `quarz doc` shows the documentation of a name.

Next, read the [documentation](@/docs/_index.md).
