The modifications made to glibc include the following:

1. Changing the System Call Mechanism: We modified the system call mechanism to route system calls through `rawposix` instead of directly invoking the kernel. The new format for making system calls is structured as:  

```
   MAKE_SYSCALL(syscallnum, "syscall|callname", arg1, arg2, arg3, arg4, arg5, arg6)
```

For each syscall file in **glibc**, we will include a header file called `syscall-template.h`. This header will contain the following content:

```c
#include <sys/syscall.h>
#include <stdint.h>    // For uint64_t
#include <unistd.h>
#include <lind_syscall.h>

// Define NOTUSED for unused arguments
#define NOTUSED 0xdeadbeefdeadbeefULL

// Macro to create a syscall and redirect it to rawposix
#define MAKE_SYSCALL(syscallnum, callname, arg1, arg2, arg3, arg4, arg5, arg6) \
    lind_syscall(syscallnum, \
                 (unsigned long long)(callname), \
                 (unsigned long long)(arg1), \
                 (unsigned long long)(arg2), \
                 (unsigned long long)(arg3), \
                 (unsigned long long)(arg4), \
                 (unsigned long long)(arg5), \
                 (unsigned long long)(arg6))
```

The `MAKE_SYSCALL` macro is designed to redirect the syscall to **rawposix**, providing an interface for making syscalls in this context.

2. Eliminating Assembly Code: Since WebAssembly (WASM) does not support assembly, all components in glibc related to assembly were removed. For inline assembly, the affected code was rewritten in C. Additionally, files ending with `.s` were converted to `.c` files, and their functionalities were reimplemented using C.

3. Handling Automatically Generated `.s` Files: glibc automatically generates `.s` files for certain system calls. To address this, the script responsible for this generation was disabled. Instead, the corresponding system calls were manually implemented in C, and the new implementations were placed in appropriate `.c` files.  
