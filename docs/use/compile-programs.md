
## Compiling a WebAssembly Module with Clang

Let's start with a simple example, `malloc-test.c`, which demonstrates dynamic memory allocation using `malloc` in C. We will compile this C program to a WebAssembly module using Clang with the WASI target.

### Example C Program: `malloc-test.c`

```c
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

int main() {
    const char *str = "Hello from Dennis's WASM!\n";

    size_t str_len = strlen(str) + 1;

    char *buf = malloc(str_len);

    if (buf == NULL) {
        return -1;
    }

    strcpy(buf, str);

    write(1, buf, str_len - 1);

    free(buf);

    return 0;
}
```

### Compiling the Program

Use the following command to compile the `malloc-test.c` program to WebAssembly:

```sh
../../clang+llvm-16.0.4-x86_64-linux-gnu-ubuntu-22.04/bin/clang-16 --target=wasm32-unknown-wasi --sysroot /home/dennis/Documents/Just-One-Turtle/wasi-libc/sysroot malloc-test.c -g -O0 -o malloc-test.wasm
```

- `--target=wasm32-unknown-wasi`: Specifies the target to be WebAssembly with WASI.
- `--sysroot /home/dennis/Documents/Just-One-Turtle/wasi-libc/sysroot`: Points to the WASI sysroot directory.
- `-g`: Includes debugging information.
- `-O0`: Disables optimizations for easier debugging.

## Frequently Used Flags
- `--target=wasm32-unknown-wasi` for compiling to wasm
- `-c` for compiling as a library without main executable
- `-pthread` then the compiler to understand `__tls_base` etc
- `--sysroot` specifying the stand library path

## Modify stubs.h
Modify the file `stubs.h` located in `/home/lind-wasm/glibc/target/include/gnu` to

```
/* This file is automatically generated.
   This file selects the right generated file of `__stub_FUNCTION' macros
   based on the architecture being compiled for.  */


//#if !defined __x86_64__
//# include <gnu/stubs-32.h>
//#endif
#if defined __x86_64__ && defined __LP64__
# include <gnu/stubs-64.h>
#endif
#if defined __x86_64__ && defined __ILP32__
# include <gnu/stubs-x32.h>
#endif
```

After modifying `stubs.h` remember to run `gen_sysroot.sh` again

```
cd /home/lind-wasm/glibc
./gen_sysroot.sh
```

Then we cd to lind-wasm-tests for testing

```
cd /home/lind-wasm/lind-wasm-tests
```

## Compile C to wasm
If you don't need to use glibc, modify `add.c` to the c file you want to compile and `add.wasm` is the wasm file you get(you can modify add to the name you want). Modify `/home/clang+llvm-16.0.4-x86_64-linux-gnu-ubuntu-22.04/bin/clang` to you compiler's path

```
/home/clang+llvm-16.0.4-x86_64-linux-gnu-ubuntu-22.04/bin/clang --target=wasm32 -nostdlib -Wl,--no-entry -Wl,--export-all -o add.wasm add.c
```

If you need to use glibc(such as printf, printf.c is located in lind-wasm/lind-wasm-tests), modify `printf.c` to the c file you want to compile and `printf.wasm` is the wasm file you get(you can modify printf to the name you want). Modify `/home/clang+llvm-16.0.4-x86_64-linux-gnu-ubuntu-22.04/bin/clang-16` to you compiler's path

```
/home/clang+llvm-16.0.4-x86_64-linux-gnu-ubuntu-22.04/bin/clang --target=wasm32-unknown-wasi --sysroot /home/lind-wasm/glibc/sysroot printf.c -g -O0 -o printf.wasm
```

You should get printf.wasm after compiling printf.c
