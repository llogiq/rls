[![Build Status](https://travis-ci.org/rust-lang-nursery/rls.svg?branch=master)](https://travis-ci.org/rust-lang-nursery/rls) [![Build status](https://ci.appveyor.com/api/projects/status/cxfejvsqnnc1oygs?svg=true)](https://ci.appveyor.com/project/jonathandturner/rls-x6grn)



# Rust Language Server (RLS)

**This project is in the early stages of development, it is not yet ready for real
use. It will probably eat your laundry.**

The RLS provides a server that runs in the background, providing IDEs,
editors, and other tools with information about Rust programs. It supports
functionality such as 'goto definition', symbol search, reformatting, and code
completion, and enables renaming and refactorings.

The RLS gets its source data from the compiler and from
[Racer](https://github.com/phildawes/racer). Where possible it uses data from
the compiler which is precise and complete. Where its not possible, (for example
for code completion and where building is too slow), it uses Racer.

Since the Rust compiler does not yet support end-to-end incremental compilation,
we can't offer a perfect experience. However, by optimising our use of the
compiler and falling back to Racer, we can offer a pretty good experience for
small to medium sized crates. As the RLS and compiler evolve, we'll offer a
better experience for larger and larger crates.

The RLS is designed to be frontend-independent. We hope it will be widely
adopted by different editors and IDEs. To seed development, we provide a
[reference implementation of an RLS frontend](https://github.com/jonathandturner/rustls_vscode)
for [Visual Studio Code](https://code.visualstudio.com/).


## Building

Since the RLS is closely linked to the compiler and is in active development,
you'll need a recent nightly compiler to build it.  In our experience, the 
nightly from rustup should be avoided for the time being.  Instead use a nightly 
you build, or one from a direct download.

Use `cargo build` to build.


## Running

You can run the rls by hand with:

```
cargo run
```

Though more commonly, you'll use an IDE plugin to invoke it for you. For this to work,
ensure that the `rls` command is in your path.

To work with the RLS, your project must be buildable using `cargo build`. If you
use syntax extensions or build scripts, it is likely things will go wrong.

If you're using recent versions of rustup, you will also need to make sure that the
compiler dynamic libraries are available for the RLS to load.  You can see where they 
are using:

```
rustc --print sysroot
```

This will show you where the compiler keeps the dynamic libs.  In Windows, this will be 
in the `bin` directory under this path.  On other platforms, it will be in the `lib`
directory.

On Windows, make sure this path (plus `bin`) is in your PATH.  For example:

```
set PATH=%PATH%;C:\Users\appveyor\.multirust\toolchains\nightly-i686-pc-windows-gnu\bin
```

For Mac, you need to set the DYLD_LIBRARY_PATH.  For example:

```
export DYLD_LIBRARY_PATH=/Users/jturner/.rustup/toolchains/nightly-x86_64-apple-darwin/lib
```

I believe Linux is the same, except that it uses LD_LIBRARY_PATH.


### Configuration

The RLS can be configured on a per-project basis by adding a file called
`rls.toml` to the project root (i.e., next to Cargo.toml). Entries in this file
will affect how the RLS operates and how it builds your project.

Currently we accept the following options:

* `build_lib` (`bool`, defaults to `false`) checks the project as if you passed the `--lib` argument to cargo.
* `cfg_test` (`bool`, defaults to `true`) checks the project as if you were running `cargo test` rather than `cargo build`. I.e., compiles (but does not run) test code.


### VSCode integration

To run with VSCode, you'll need a recent version of that
[installed](https://code.visualstudio.com/download).

You'll then need a copy of our [VSCode extension](https://github.com/jonathandturner/rustls_vscode).

The RLS can operate via the [Language Server protocol](https://github.com/Microsoft/language-server-protocol/blob/master/protocol.md).

VSCode will start the RLS for you. Therefore to run, you just need to open the VSCode extension and run it. 
However, you must install the rls in your path so that the RLS can find it.

Be sure to provide an RLS_ROOT environment variable.  Point this variable at the root of the RLS checkout:

```
export RLS_ROOT=/Source/rls
```

## Testing

Test using `RUST_TEST_THREADS=1 cargo test`.

Testing is unfortunately minimal. There is support for regression tests, but not
many actual tests exists yet. There is signifcant [work to do](https://github.com/jonathandturner/rustls/issues/12)
before we have a comprehensive testing story.


## Standard library support

Getting the RLS to work with the standard libraries takes a little more work, we
hope to address this in the future for a more ergonomic solution (https://github.com/jonathandturner/rustls/issues/88).

The way it works is that when the libraries are built, the compiler can emit all
the data that the RLS needs. This can be read by the RLS on startup and used to
provide things like type on hover without having access to the source code for
the libraries.

The compiler gives every definition an id, and the RLS matches up these ids. In
order for the RLS to work, the id of a identifier used in the IDE and the id of
its declaration in a library must match exactly. Since ids are very unstable,
the data used by the RLS for libraries must match exactly with the crate that
your source code links with.

You need a version of the above data which exactly matches the standard
libraries you will use with your project. You can do this either by downloading
matching data, or by building your own std libs and emitting the data at the
same time.

### Download the libs

You must be using nightly, find out what date nightly you have. Note that this
may not be the date given by `--version` (build date vs distribution date). The
easiest way to do this is to download a specific day's nightly and use that.
Then, navigate to the [Rust archives](https://static.rust-lang.org/dist/index.html)
and click through to your Rust's date. You will see a lot of files. Find one
that looks like `rust-analysis-nightly-$YOUR_TARGET_TRIPLE.tar.gz`, and
download it. For example, if you are on regular Linux and have a compiler for
21st December 2016, you will want `https://static.rust-lang.org/dist/2016-12-21/rust-analysis-nightly-x86_64 -unknown-linux-gnu.tar.gz`.

OK, now open the archive. Navigate through the various sub-directories to find
one called `analysis`. It will be somewhere like: `rust-analysis-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/analysis`.
You must extract the JSON files in that `analysis` directory to `lib/save-analysis`
in your Rust sysroot, e.g., (on Linux, no multi-rust/rustup) `/usr/local/lib/save-analysis`.
To find your sysroot you can use `rustc --print=sysroot`. Note that if you change
Rust installation (e.g., using rustup), your sysroot might change.


### Build it yourself

In your Rust directory, you want to run the following:

```
# Or whatever -j you usually use.
RUSTFLAGS_STAGE2='-Zsave-analysis-api' make -j6
```

Then go get a coffee, possibly from a cafe on the other side of town if you have
a slower machine.

If all goes well, you should have a bunch of JSON data in a directory like
`~/rust/x86_64-unknown-linux-gnu/stage2/lib/rustlib/x86_64-unknown-linux-gnu/lib/save-analysis`.

You need to copy all those files (should be around 16) into a directory called
`analysis` in your Rust sysroot (see above for details).

Finally, to run the RLS you'll need to set things up to use the newly built
compiler, something like:

```
export RUSTC="/home/ncameron/rust/x86_64-unknown-linux-gnu/stage2/bin/rustc"
```

Either before you run the RLS, or before you run the IDE which will start the
RLS.


## Implementation overview

The goal of the RLS project is to provide an awesome IDE experience *now*. That
means not waiting for incremental compilation support in the compiler. However,
Rust is a somewhat complex language to analyse and providing precise and
complete information about programs requires using the compiler.

The RLS has two data sources - the compiler and Racer. The compiler is always
right, and always precise. But can sometimes be too slow for IDEs. Racer is
nearly always fast, but can't handle some constructs (e.g., macros) or can only
handle them with limited precision (e.g., complex generic types).

The RLS tries to provide data using the compiler. It sets a time budget and
queries both the compiler and Racer. If the compiler completes within the time
budget, we use that data. If not, we use Racer's data.

We link both Racer and the compiler into the RLS, so we don't need to shell out
to either (though see notes on the build process below). We also customise our
use of the compiler (via standard APIs) so that we can read modified files
directly from memory without saving them to disk.

### Building

The RLS tracks changes to files, and keeps the changed file in memory (i.e., the
RLS does not need the IDE to save a file before providing data). These changed
files are tracked by the 'Virtual File System' (which is a bit of a grandiose
name for a pretty simple file cache at the moment, but I expect this area to
grow significantly in the future). The VFS is in a [separate
crate](https://github.com/nrc/rls-vfs).

We want to start building before the user needs information (it would be too
slow to start a build when data is requested). However, we don't want to start a
build on every keystroke (this would be too heavy on user resources). Nor is
there any point starting multiple builds when we would throw away the data from
some of them. We therefore try to queue up and coalesce builds. This is further
documented in [src/build.rs](src/build.rs).

When we do start a build, we may also need to build dependent crates. We
therefore do a full `cargo build`. However, we do not compile the last crate
(the one the user is editing in the IDE). We only run Cargo to get a command
line to build that crate. Furthermore, we cache that command line, so for most
builds (where we don't need to build dependent crates, and where we can be
reasonably sure they haven't changed since a previous build) we don't run Cargo
at all.

The command line we got from Cargo, we chop up and feed to the in-process
compiler. We then collect error messages and analysis data in JSON format
(although this is inefficient and [should
change](https://github.com/jonathandturner/rustls/issues/25)).

### Analysis data

From the compiler, we get a serialised dump of its analysis data (from name
resolution and type checking). We combine data from all crates and the standard
libraries and combine this into an index for the whole project. We cross-
reference and store this data in HashMaps and use it to look up data for the
IDE.

Reading, processing, and storing the analysis data is handled by the
[rls-analysis crate](https://github.com/nrc/rls-analysis).

### Communicating with IDEs

The RLS communicates with IDEs via
the [Language Server protocol](https://github.com/Microsoft/language-server-protocol/blob/master/protocol.md).

The LS protocol uses JSON sent over stdin/stdout. The JSON is rather dynamic -
we can't make structs to easily map to many of the protocol objects. The client
sends commands and notifications to the RLS. Commands must get a reply,
notifications do not. Usually the structure of the reply is dictated by the
protocol spec. The RLS can also send notifications to the client. So for a long
running task (such as a build), the RLS will reply quickly to acknowledge the
request, then send a message later with the result of the task.

Associating requests with replies is done using an id which must be handled by
the RLS.


### Extensions to the Language Server Protocol

The RLS uses some custom extensions to the Language Server Protocol. Currently
these are all sent from the RLS to an LSP client and are only used to improve
the user experience by showing progress indicators.

* `rustDocument/diagnosticsBegin`: notification, no arguments. Sent before a
  build starts and before any diagnostics from a build are sent.
* `rustDocument/diagnosticsEnd`: notification, no arguments. Sent when a build
  is complete (successfully or not, or even skipped) and all post-build analysis
  by the RLS is complete.


## Contributing

The RLS is open source and we'd love you to contribute to the project. Testing,
reporting issues, writing documentation, writing tests, writing code, and
implementing clients are all extremely valuable.

Here is the list of known [issues](https://github.com/jonathandturner/rustls/issues).
These are [good issues to start on](https://github.com/jonathandturner/rustls/issues?q=is%3Aopen+is%3Aissue+label%3Aeasy).

We're happy to help however we can. The best way to get help is either to
leave a comment on an issue in this repo, or to ping us (nrc or jntrnr) in #rust-tools
on IRC.

We'd love for existing and new tools to use the RLS. If that sounds interesting
please get in touch by filing an issue or on IRC.
