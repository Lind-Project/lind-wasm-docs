The modifications made to glibc include the following:

1. Changing the System Call Mechanism: We modified the system call mechanism to route system calls through `rawposix` instead of directly invoking the kernel. The new format for making system calls is structured as:  

```
   MAKE_SYSCALL(syscallnum, "syscall|callname", arg1, arg2, arg3, arg4, arg5, arg6)
```

2. Eliminating Assembly Code: Since WebAssembly (WASM) does not support assembly, all components in glibc related to assembly were removed. For inline assembly, the affected code was rewritten in C. Additionally, files ending with `.s` were converted to `.c` files, and their functionalities were reimplemented using C.

3. Handling Automatically Generated `.s` Files: glibc automatically generates `.s` files for certain system calls. To address this, the script responsible for this generation was disabled. Instead, the corresponding system calls were manually implemented in C, and the new implementations were placed in appropriate `.c` files.  
