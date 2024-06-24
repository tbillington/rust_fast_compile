# Minimising Rust compile times

This repo demonstrates techniques to reduce the time taken to produce a Rust binary. They're collected from various blog posts and conversations I've had. I take no ownership over them, I just wanted to compile them in one place for my own convenience.

Inspired by [min-sized-rist](https://github.com/johnthagen/min-sized-rust/tree/main).

## Project agnostic

Changes that could apply to any Rust project that don't require significant changes to the project structure or source code.

### Only compile if you need to

Cargo has the [`check`](https://doc.rust-lang.org/cargo/commands/cargo-check.html) command to verify if the program would build, still giving warnings & error messages, but without the overhead of running codegen.

Great for use in CI if you just need to confirm if a certain commit would build or not.

### Use a faster linker

Use [mold](https://github.com/rui314/mold) on linux, or [lld](https://lld.llvm.org/) on windows. You can use mold on mac, however the new linker ld-prime [introduced in xcode 15](https://developer.apple.com/documentation/xcode-release-notes/xcode-15-release-notes#Linking) gets quite close.

If you'd like to try a more experimental linker, [wild](https://github.com/davidlattimore/wild) is available on linux, and ran about 25% faster than mold in my project.

### Have better hardware

Rust compilation is resource intensive, and better hardware will go a long way. In my anecdotal evidence the most benefits will come from a stronger CPU, and making sure you have enough memory that you don't encounter page faults.

From there, spare memory for the OS to cache files helps, then a faster drive.

### Try the parallel front-end

In Rust nightly there is a parallel front-end compiler. You can try it using [these instructions](https://blog.rust-lang.org/2023/11/09/parallel-rustc.html#how-to-use-it).

### Disable incremental compilation

If you're doing from scratch builds, commonly used in CI pipelines, you can disable incremental compilation to avoid it's overhead.

```toml
[profile.ci]
incremental = false
```

or

```sh
ENV CARGO_INCREMENTAL=0
```

### Try the Cranelift backend

Cranelift can often compile debug builds faster than LLVM, though it currently requires nightly and is described as an "unstable" option. https://github.com/rust-lang/rustc_codegen_cranelift.

### Use Dev Drive on Windows

[Windows Dev Drives](https://devblogs.microsoft.com/visualstudio/devdrive/) use a different filesystem, and with changes to Windows Defender settings it provides a 25% average reduction in compile times.

### Reduce debug info

Reducing the [debug level](https://doc.rust-lang.org/cargo/reference/profiles.html#debug) can reduce the amount of information the compiler needs to keep track of, and how much work the linker has to do.

```toml
[profile.dev]
debug = 1 # Or 0, "line-tables-only", etc
```

### Try generic sharing

Can allow crates to share monomorphized generic code. Requires nightly Rust.

```toml
# .cargo/config.toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-Zshare-generics=y"]
```

### Compile macros with optimisations

This may speed up macro expansion, mostly useful in incremental rebuilds.

```toml
[profile.dev.build-override]
opt-level = 2

[profile.release.build-override]
opt-level = 2
```

## Project involved

Changes that require modifications to a projects dependencies, source code, or structure.

### Reducing how much code you compile

Removing unused dependencies.

Disabling unused features on dependencies.

Choosing alternative dependencies that aren't as heavy if you don't need all the features.

### Splitting your project into different crates

Rustc uses crates as the unit of compilation. To take advantage of parallelism of crate compilation split your crate into multiple crates that depend only on their subset of dependencies, then consume those from your "main" crate.

This will not only allow running more compilation in parallel, but can also allow certain crates to start compiling earlier in the build process.

You can do this by utilising cargo [workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html), or by creating separate cargo crates.

## Supplemental options

Further avenues you can explore.

### Understand the Cargo profiles

You can build a good model of what options are available to you by understanding the configuration of the build in [Cargo profiles](https://doc.rust-lang.org/cargo/reference/profiles.html), and how you could modify them to suit your needs.

### Visualise build times

You can visualise the build with a flag to Cargo.

```sh
cargo build --timings
```

## Sources

https://matklad.github.io/2021/09/04/fast-rust-builds.html

https://xxchan.me/cs/2023/02/17/optimize-rust-comptime-en.html

https://benw.is/posts/how-i-improved-my-rust-compile-times-part2

https://bevyengine.org/learn/quick-start/getting-started/setup/#enable-fast-compiles-optional
