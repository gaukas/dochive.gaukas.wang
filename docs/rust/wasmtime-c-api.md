---
layout: default
title: Extend wasmtime C API for wasmtime-go
parent: Rust
nav_order: 9
---

# Extend wasmtime C API for wasmtime-go

Recently I have been working on [wasmtime](https://github.com/bytecodealliance/wasmtime) C API, trying to add some features that exists in its Rust API but not in C API, where the latter is supporting other languages' bindings, e.g., [wasmtime-go](https://github.com/bytecodealliance/wasmtime-go). I found the online resources aren't too helpful and it took me sometime to figure it out how to test the Windows build locally. 

## Requirements
- __cargo 1.72.0__ or later
- __rustc 1.72.0__ or later
- __go 1.20__ or later
- A clone of wasmtime repository
- A clone of wasmtime-go repository

## Develop wasmtime C API for different platforms

The feature I am developing is unfortunately very platform-specific, so I have to conditionally compile its implementation for different platforms.

And basically this is what I did: 

```rust
use std::ffi::c_void;
use std::fs::File;
#[cfg(unix)]
use std::os::fd::FromRawFd;
#[cfg(windows)]
use std::os::windows::{io::FromRawHandle, raw::HANDLE};
```

Later in the code, I also had to declare the same function for three times: 

```rust 
#[cfg(unix)]
#[no_mangle]
pub unsafe extern "C" fn wasmtime_context_insert_file(
    context: CStoreContextMut,
    guest_fd: u32,
    host_fd: *mut c_void, // i32
    access_mode: u32,
) {
    // ...
}

#[cfg(windows)]
#[no_mangle]
pub unsafe extern "C" fn wasmtime_context_insert_file(
    context: CStoreContextMut,
    guest_fd: u32,
    host_fd: *mut c_void, // HANDLE
    access_mode: u32,
) {
    // ...
}

#[cfg(not(any(unix, windows)))]
#[no_mangle]
pub unsafe extern "C" fn wasmtime_context_insert_file(
    context: CStoreContextMut,
    guest_fd: u32,
    host_fd: *mut c_void, // i32
    access_mode: u32,
) {
    unimplemented!("wasmtime_context_insert_file")
}
```

Since I managed to make the signature for the function the same for all platforms, the C header file will require only one single function declaration. That's lucky, because I sincerely don't know how to declare a function with different signatures for different platforms in C header file -- I don't know what's the correct `#tag` for that.

```c
WASM_API_EXTERN void wasmtime_context_insert_file(wasmtime_context_t *context, uint32_t guest_fd, void *host_fd, uint32_t access_mode);
```

## Compiling

It is quite straightforward to compile the library for different platforms. Or... at least it looks like it. 

{: .warning }
> DON'T TRY THIS AT HOME, UNLESS YOU HAVE FINISHED READING THIS ARTICLE.

### Linux
Compile for release: 
```bash
cargo build --release -p wasmtime-c-api
```

It produces in `wasmtime/target/release` a `libwasmtime.a`, which is the static library for linux/macOS platforms.

If you want to compile for debug, you can do: 
```bash
cargo build -p wasmtime-c-api
```

The compilation output will be in `wasmtime/target/debug`.

### Windows 

The compilation command is the same as Linux. 

```bash
cargo build --release -p wasmtime-c-api
```

It produces in `wasmtime/target/release` three target files:
- `wasmtime.dll`
- `wasmtime.dll.lib`
- `wasmtime.lib`

Just as the content in the pre-compiled release package, `wasmtime-{version}-x86_64-windows-c-api.zip`.

## Testing

It is non-trivial to test C API with only itself. You will need a caller. Therefore I will test it with wasmtime-go. Anyways the changes I am contributing to wasmtime C API is for the sake of wasmtime-go, so it makes sense to test it with wasmtime-go.

wasmtime-go is released with compiled `wasmtime` libraries for different platforms, and C header files. However, since we manually compiled them, we will need to manually put them into the correct place.

### Linux

Just run `wasmtime-go/ci/local.sh`. 

```bash
./ci/local.sh /path/to/your/fork/of/wasmtime
```

It will copy `libwasmtime.a` from `wasmtime/target/release` (or `wasmtime/target/debug` if release doesn't exist) to `wasmtime-go/build/{os-arch}` (for example, `wasmtime-go/build/linux-x86_64`), along with _linking_ (not copying, note this) all the header files (`.h`) in `wasmtime/crates/c-api/include` to `wasmtime-go/build/include`.

This should work for macos as well, but I haven't tested it. You may sponsor me a macbook if you want me to test it on macos ;)

### Windows 

This is the tricky part. 

You will need to manually copy `wasmtime.dll`, `wasmtime.dll.lib` and `wasmtime.lib` to `wasmtime-go/build/windows-x86_64`. And C header files to `wasmtime-go/build/include`.

Now things seems to be on the same page. 

Run `go test` with your implemented test, boom: 

```
corrupt .drectve at end of def file
```

You will get a bunch of error messages and the beginning of the error message is `corrupt .drectve at end of def file`. What does that even mean? 

Let's check what's in the released `wasmtime-go`, and what does `wasmtime-go/ci/download-wasmtime.py` pull on Pull Requests. 

In the released `wasmtime-go` (v12.0.0), I see only `libwasmtime.a`. And the Pull Request CI actually downloads `wasmtime-{version}-x86_64-mingw-c-api.zip` from `wasmtime`, which includes 3 files: 
- `libwasmtime.a`
- `libwasmtime.dll.a`
- `wasmtime.dll`

This isn't good. I do not have `libwasmtime.dll.a` in our local build. What happened? So it turns out that my default toolchain is from `msvc`, but the CI is using `mingw`, or in other name, `gnu`. 

#### Install the new toolchain

```bash
rustup toolchain install stable-x86_64-pc-windows-gnu
```

And in `.rustup/settings.toml`, change the default toolchain to `stable-x86_64-pc-windows-gnu`:

```toml
default_toolchain = "stable-x86_64-pc-windows-gnu"
```

#### Compile with the new toolchain

Again

```bash
cargo build --release -p wasmtime-c-api
```

It produces in `wasmtime/target/release` three target files:
- `libwasmtime.a`
- `libwasmtime.dll.a`
- `wasmtime.dll`

#### Copy the new files to wasmtime-go

Now copy `libwasmtime.a`, `libwasmtime.dll.a` and `wasmtime.dll` to `wasmtime-go/build/windows-x86_64`. And C header files to `wasmtime-go/build/include`. 

#### Run the test

Run `go test` with your implemented test. Then if everything goes well, you will get 

```
exit status 0xc0000135
```

WHY? Okay, it turns out the `wasmtime.dll` we compiled cannot be found by `wasmtime-go`. Adding the path to `wasmtime.dll` to `PATH` environment variable solves the problem.

Remember to restart your terminal! 

And then `go test` finally passes. 

#### Side note: release with only `libwasmtime.a`
The `wasmtime-go` is released with only `libwasmtime.a` not even any dll. However when I first put only `libwasmtime.a` in `wasmtime-go/build/windows-x86_64`, `go test` will pass but produce multiple warnings including 

```
corrupt .drectve at end of def file
```

Haven't figured out why yet. It seems to be an issue with the linker for CGO. 