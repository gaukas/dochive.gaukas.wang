---
title: Compile Rust library to Android targets
categories:
- tutorial
- programming
tags: 
- linux
- rust
- android
date: 11/06/2023 21:00
---
I noticed there was no good tutorial online about how to compile Rust library to Android target. So here's one after a few hours of struggling.

## Requirements
- __cargo__
- __rustc__
- Android NDK

## Download Android NDK

To start with, Android NDK is required. It could be downloaded from here: https://developer.android.com/ndk/downloads

We are going to show the steps for Linux in this tutorial.  

```bash
wget https://dl.google.com/android/repository/android-ndk-r26b-linux.zip
unzip android-ndk-r26b-linux.zip
```

To compile Rust code into Android targets, we need the toolchain shipped with Android NDK located at `android-ndk-r26b/toolchains/llvm/prebuilt/linux-x86_64/bin`. This directory should be added to `PATH` environment variable.

```bash
export PATH=$PATH:/path/to/android-ndk-r26b/toolchains/llvm/prebuilt/linux-x86_64/bin
```

## Compile Rust Library

To compile Rust library to Android, we need two things from the toolchain: `ar` and `clang`. Where `ar` is target architecture/version-independent and `clang` is architecture/version-dependent.

Take `aarch64` as an example, Rust will try to run `aarch64-linux-android-ar` and `aarch64-linux-android-clang` to compile the library. So we need to make sure these two commands are available in `PATH`. From the Android NDK, we can find no `aarch64-linux-android-ar` but only `llvm-ar`. So we need to create a soft link to `llvm-ar` or copy it to `aarch64-linux-android-ar`.

```bash
cp /path/to/llvm-ar /path/to/aarch64-linux-android-ar
```

In terms of the `clang`, we can specify it in `~/.cargo/config.toml`. The content of the file should be like this:

```toml
[target.aarch64-linux-android]
linker = "aarch64-linux-android34-clang"

[target.x86_64-linux-android]
linker = "x86_64-linux-android34-clang"

[target.i686-linux-android]
linker = "i686-linux-android34-clang"

[target.armv7a-linux-androideabi]
linker = "armv7a-linux-androideabi34-clang"
```

There's one more thing to do before we could compile the library: we need to add the target to Rust. For example, if we want to compile the library to `aarch64-linux-android`, we need to run the following command:

```bash
rustup target add aarch64-linux-android
```

Now we are ready to compile the library. The command is as simple as:

```bash
cargo build --target=aarch64-linux-android --release --manifest-path /path/to/Cargo.toml
```

And it should work magically!

## Troubleshooting

### error occurred: Command "aarch64-linux-android-clang" ... Permission denied

Basically it says it cannot run `aarch64-linux-android-clang`. You might want to double check you specified the correct linker and it is in `PATH`.

### error occurred: Command "aarch64-linux-android-ar" ... Permission denied

Basically it says it cannot run `aarch64-linux-android-ar`. You might want to double check you copied `llvm-ar` to `aarch64-linux-android-ar` and it is in `PATH`.

As of why they are `Permission denied` not `Command not found`, I have no idea. 