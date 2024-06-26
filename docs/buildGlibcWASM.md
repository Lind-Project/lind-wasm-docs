# Building glibc to WASM

I'd recommend reading this doc in its entirety before trying to compile.

## Prerequisites
We need WASM compatible `clang` and `ar`, which can be built locally from `wasi-sdk`
[https://github.com/WebAssembly/wasi-sdk](https://github.com/WebAssembly/wasi-sdk)

Also strongly recommend to install `wasm-objdump` from the `wabt` toolkit
[https://github.com/WebAssembly/wabt](https://github.com/WebAssembly/wabt)

## Configure
Firstly we should write and run a config script like this in the glibc root directory

```
#!/bin/bash
set -e
BUILDDIR=build
mkdir -p $BUILDDIR
cd $BUILDDIR
../configure --disable-werror --disable-hidden-plt --with-headers=/usr/i686-linux-gnu/include --prefix=/sysroot-coulson --host=i686-linux-gnu --build=i686-linux-gnu\
    CFLAGS=" -O2 -g" \
    CC="/wasi-sdk/build/wasi-sdk-22.0/bin/clang-18 --target=wasm32-unkown-wasi -v -Wno-int-conversion"
```
You must replace `CC` to the path to your `clang`. If you define `BUILDDIR=build`, then the compiled WASM object files will appear under `glibc/build`.
Be aware that you should make sure this build directory is empty before running config script, so you need to `rm -rf build` before recompiling it.

A crutial job of the configure script is deciding which sysdeps directories to use according to the `host` and `build` string.
We already changed the configure script in glibc root directory, and the lind add-on directories are already baked to be included.

The configure flags we need:

- `disable-werror`: we have countless warnings, so we ignore them for now
- `disable-hidden-plt`: PLT bypassing optimization is causing ~50k errors, simply disable it for now
- `with-headers`: glibc requires Linux kernel headers to be installed before config and compile, so set this flag to a built-in sysroot of 32bit, this doesn't seem to raise an issue for our WASM built
- `prefix=`: this is the path of the generated sysroot when you use `make install`. But note that, the glibc's `make install` will NOT work at all for WASM, because WASM sysroot has differen structure convention, also requires an `llvm-ar` arhive. More details, see my script `gen_sysroot.sh`. However, we can still use `make install` just to generate the `.h` files of the sysroot
- `host` & `target`: we start off from the sysdeps direcotries of i686, so fixing these options

The compiler flags we need:

- `-O2 -g`: nah the glibc won't allow you to compile with `O0`, so we bear with this `O2` optimization during debugging. But sometimes you can change to `O1`.
-  `-Wno-int-conversion`: we disable int conversion warnings, cuz all 32bit types as WASM function arguments, are eventually i32 anyway
- `--target=wasm32-unkown-wasi`: this tells the compiler we want to compile to WASM

After config succeed, you will see these in the `build` directory,

```
Makefile  bits  config.h  config.log  config.make  config.status
```

## Compiling to object files

In the build directory, usually we use `make --keep-going -j$(nproc)`. The first flag is to continue compiling after errors, we need this cuz there are too many errors now (mainly due to assembly about threading). The `-j` is important to speed it up, but also makes the compilation log interleaved. The compilation log is **VERY IMPORTANT**, which tells why a given c file failed to be compiled. So sometimes we don't want the `-j`. Also, we can copy the actual compiler command in the compile log. For such commands, if we want to compile a single C file, only the source file path need to be further specified. We can use this to test compiling a specific file.

## Generating WASM sysroot
This procedure is specified in the `gen_sysroot.sh` script in our glibc repo. It's main job is to generate a WASM sysroot structre like

```
sysroot/
- include/
  - wasm32-wasi/
    - stdio.h
    - ...other headers
- lib/
  - wasm32-wasi/
    - crt1.o
    - libc.a
```
Note that the header files should be pre-generated using `make install`. The crt1.o should be pre-compiled from this simple C file (see the WASM compile doc as well). The main job of this script is find every valid WASM `.o` file in the `build` directory, and group everything into `libc.a`, an `llvm-ar` arvhive.

```
void _start() {
    main();
}
void __wasm_call_dtors() {}
void __wasi_proc_exit(unsigned int exit_code) {}
```
Here are some macros we need to twist:

- `src_dir`: the glibc `build` directory that contains all the WASM object files
- `include_source_dir`: the path to your pre-built headers
- `crt1_source_path`: path to your pre-built crt1.o
- `lind_syscall_path`: you also need to pre-compile `lind_syscall.o`, just like `crt1.o`, and the source file is under glibc/lind_syscall
- `sysroot_dir`: path to generate the sysroot at
- `output_archive`: the path to the generate the libc.a, should be align with `sysroot_dir`

## Running only the pre-processor

The pre-processing stage of the compiler expland all `#include` and all macros. Because recursive macros are so prevalent in glibc, sometimes you want to see the actual source file after epxansions, then you want to use the `-E` option of `clang`.

The easiest way is to copy the compiler command from the compile log and add a `-E`. If you want to run pre-prosessor on ALL files, then run the configure script again, but before compiling, add `-E` to `config.make` right after

```
# Build tools.
CC = /wasi-sdk/build/wasi-sdk-22.0/bin/clang-18 [ADD HERE!]
```