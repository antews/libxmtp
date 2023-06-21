# Uniffi-based bindings for XMTP v3

This crate provides cross-platform Uniffi bindings for XMTP v3.

# Status

- Android is tested end-to-end via an example app in `../examples/android`.
- iOS has not been tested.

# Consuming this crate

The generated artifacts of this crate are the bindings interface (`xmtpv3.kt`) generated by `uniffi`, and the cross-compiled binaries (`jniLibs/`) generated by `cross`.

- Run `./setup_android_example.sh` to copy these artifacts into the example Android app. Alternatively, modify the script to set up an app of your choice.
- Open the `build.gradle` of the example Android app in Android Studio.

# Rebuilding this crate

The generated bindings (`xmtp_dh.kt`) and the cross-compiled binaries (`jniLibs`) have been committed alongside the crate so you do not need to rebuild unless you make changes. The build is very slow (~3 mins on incremental builds, ~30 mins on full builds, per-target). Future changes will simplify the process and improve the build time as well as setting up async builds in CI.

- Run `./gen_kotlin.sh` to re-generate the bindings
- Install Docker
- Install Cross for zero setup cross-platform builds: `cargo install cross --git https://github.com/cross-rs/cross`
- Run `./cross_build.sh` to cross-compile (this is SLOW)

`Cross` allows us to run cross-platform builds without needing to download all of the relevant toolchains and should work regardless of your host OS. It is possible that the build time can be improved by building natively without Cross.

# Running tests

Ensure a local API host is running - run `dev/up` from the repo root.

You'll need to do the following one-time setup to run Kotlin tests:

- Run `brew install kotlin` to get `kotlinc`
- Install the JRE from `www.java.com`
- Run `make install-jar` and add both jars to your CLASSPATH environment variable, for example add `export CLASSPATH=$HOME/jna/jna.jar:$HOME/jna/kotlinx-coroutines-core-jvm.jar` to your `.zshrc`

If you want to skip the setup, you can also run `cargo test -- --skip kts` to only run Rust unit tests. CI will run all tests regardless.

# Uniffi

We are using Uniffi with the latest procedural macros syntax where possible, which also gives us async support. It is important to learn the syntax: https://mozilla.github.io/uniffi-rs/proc_macro/index.html

For the most part, any mistakes in the Uniffi interface will manifest as a compile error when running `./gen_kotlin.sh`. Some details are described below so that they are easier to understand.

## Object Lifetimes

Any objects crossing the Uniffi interface boundary must be wrapped in `Arc<>`, so that Uniffi can marshall it back and forth between [raw pointers](https://mozilla.github.io/uniffi-rs/internals/object_references.html#lifetimes) before passing it to the foreign language. The usage of Arc means that we do not need to manually destroy objects on the Rust side, however depending on the target platform, the foreign language may need to automatically or manually [release the pointer back to Rust](https://mozilla.github.io/uniffi-rs/kotlin/lifetimes.html) when done.

## Async and concurrency

We use Tokio as our multi-threaded [async runtime](https://rust-lang.github.io/async-book/08_ecosystem/00_chapter.html). Uniffi can use this runtime on async methods and objects using the annotation `#[uniffi::export(async_runtime = ‘tokio’)]`. Uniffi plumbs up an executor (scheduler) in the foreign language to the Tokio runtime in Rust. More details [here](https://github.com/mozilla/uniffi-rs/blob/734050dbf1493ca92963f29bd3df49bb92bf7fb2/uniffi_core/src/ffi/rustfuture.rs#L11-L18).

Because the foreign language may be multi-threaded, any objects passed to the foreign language must be `Send` and `Sync`, and [no references to `&mut self` are permitted](https://mozilla.github.io/uniffi-rs/udl/interfaces.html#concurrent-access). TODO: interior mutability pattern we should use, sync vs async mutex/rwlock