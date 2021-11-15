An implementation of "ML-Like Lambda Calculus" (mlllc, or m3lc, hence my
not-so-clever name for source files). This document describes the features of
the library and some notes on its usage; for details on implementation, see
comments in the relevant source files.

## TL;DR

`cargo run --release --quiet -- examples/fibbit.m3lc`

## The Language

A `m3lc` file consists of a series of definitions followed by a main. A definition is a name, followed by `:=`, followed by a term, followed by a `;`. A main is either a definition whose name is `main`, or a term (with no closing `;`). A term is a lambda calculus term with function abstraction `fn x => t` and juxtaposition application `x y`.

For the formal grammar, see `src/m3lc.pest`.

## Cargo

As promised, here's a whirlwind tour through running and building Rust code
with `cargo`, the Rust build system. All commands can be run anywhere in the
project directory: cargo searches up the directory tree for a `Cargo.toml`
file, which represents the root of the package.

By default (as in this codebase) Cargo builds a crate (the Cargo jargon for
library) from `src/lib.rs` and a binary from `src/main.rs`. Convention is to
put almost all code in the library, as here.

To build the executable, run `cargo build`. To build it with release
optimizations (the executable will be _much_ - an order of magnitude - faster;
this difference is not uncommon for Rust code), run `cargo build --release`. By
default, these put a binary at `target/debug/[binary_name]` and
`target/release/[binary_name]`, respectively. The default binary name is the
crate name, i.e. `m3lc` in this case.

To build and run tests, run `cargo test`. To build and run benchmarks in
release mode, run `cargo bench`.

To build and run the executable, run `cargo run` or `cargo run --release`. To
pass arguments to the executable, place them after a `--`, i.e.
`cargo run --release -- examples/fibbit.m3lc`. To mute compilation output, pass
`--quiet` to cargo. Cargo is stateless, so it needs to recompile each time, but
by default this recomilation will be incremental, i.e. if you haven't changed
source files it will be very very very fast.

To build documentation, run `cargo doc`. This places the html documentation,
autogenerated from doc comments (`///` and `//!`), at
`target/doc/[library_name]/index.html`. Any examples in the documentation are
run as unit tests by `cargo test`.

## Rust Nightly

`m3lc` depends on a nightly Rust toolchain, and in particular three unstable
features:

1. `box_patterns` allows pattern-mathing on boxed types. It's used in a lot of
   places, because boxed types are common in this code (because the `Term` type
   is recursive) and I find it more ergonomic than calling `as_ref` or
   dereferencing everywhere.
2. `box_syntax` allows `box expr` instead of `Box::new(expr)`. This is also
   mostly an ergonomic thing for highly-boxed code. (There are also some subtle
   performance implications of this, since it doesn't require intermediate
   stack allocation.)
3. `test`, which creates an easy-to-use benchmarking harness on top of the Rust
   testing API, used for benchmarking reduction code.

Neither of these are at all necessary, but bringing in unstable features in
non-production code isn't costly at all, these features are just mildly useful
and the Rust team likes to ask people to do it where appropriate so they can
get bug testers and feedback on APIs. The `rust-toolchain.toml` file will make
`Cargo` use nightly by default. You can override this setting by passing
`+[toolchain]` directly to Cargo, for instance, `cargo +stable build` (though
this crate won't build without nightly).

Let me know if you want this to build on stable. The nightly features are just
ergonomics and because I like to try to be a good open source citizen; it's a
super easy change to be compatible with stable, and if this was anything I
expected anyone to use it'd be on stable already.

## Architecture

Each piece of the assignment corresponds (roughly) to one of the source files.

1. The parser: this is handled by the `parse.rs` file and the `m3lc.pest`
   grammar. It's built on top of the `pest` and `pest_consume` crates ("crate"
   is the Rust/Cargo term for "library"), which together allow the strongly
   typed, recursive pattern-matching on terms that you see consumed by the
   `pest_consume::parser` macro.
2. The term: the Rust types corresponding to the AST for a `.m3lc` file is
   defined in the `grammar.rs` file. This file is fairly self-explanatory. It
   does two-pieces of non-trivial work. First, it "unrolls" a `File` into a
   `Term` via applying defined terms, in reverse order, as lambda abstractions
   over the `main` term. Second, it implements `Display` for the AST types,
   which allows printing them as readable `.m3lc` code.
3. Reduction: the `reduce.rs` file implements the public methods `reduce` and
   `alpha_equiv` on `Term`, which perform (normal order) beta-reduction and
   check alpha-equivalence, respectively. (Alpha-equivalence was not strictly
   part of the assignment, but is useful for automated testing of normal order
   beta-reduction, which only guarantees normalization up to alpha-renaming.)
4. Testing: the simple tests are implemented as unit tests in the various
   files. A common idiom here is to use a Rust macro to allow writing test code
   generic over the term to be tested. The longer-form tests are in the
   `examples/` directory and are not tested automatically.
5. Specific terms: each term is implmented in a separate file in the
   `examples/` directory in the project root.
6. Debug mode: this is implemented as a `-v` (for verbose) flag for the CLI,
   parsed by the `structopt` crate in `cli.rs`. For full documentation of the
   CLI, pass the `-h` flag.

## Extras

There are a couple of "extra" things I implemented beyond the problem
assignment; I figured I'd document them here:

1. Alpha-equivalence: as mentioned, this is useful for unit testing, but is
   also critical for #2.
2. Term inference: unless given the `-n` command-line flag, the CLI checks for
   alpha-equivalence of the final output to boolean and church numeral types.
   This work is handled by the `guess_val` method in `cli.rs`, which relies on
   the types defined in the `data/` source directory.
3. Syntax highlighting: I like looking at pretty colors, so I wrote a super
   simple [neo]vim syntax plugin. To enable it, copy or symlink `vim/` to
   `${XDG_DATA_HOME:~/.local/share}/nvim/site/pack/m3lc/` for neovim or
   `~/.vim/pack/m3lc/` for mainline vim.
4. Performance: originally, I implemented this very lazily without paying any
   attention to performance (I was using Rust for its type system, not for
   performance). Then it turned out Ryan and Zach's javascript implementation
   was faster than my Rust implementation because I was just cloning everywhere
   so I didn't have to worry about the borrow checker (which isn't always
   friendly to recursive types), a common theme in lazy Rust code. But
   javascript code being faster than Rust code was obviously unacceptable to my
   Rust-obsessed brain, so I did a bunch of performance optimization, and now
   this code is quite fast IMO. That's mostly in the reduction code and is
   documented there. Interestingly, the remaining runtime (about 2/3 of the
   typical runtime of the program) is in the `get_fresh_ident` method, which
   has to do a bunch of messy stuff with copying around strings.
