# Memory Management and Vmmap

## What is a Vmmap?

A vmmap is a tool for managing a process’s memory layout within an operating system. It provides detailed insights into allocated memory regions, including the heap, stack, and memory-mapped files. Additionally, it displays access permissions (read-only, read-write, executable), memory region boundaries, sizes, and any mapped files.
Motivation

## Motivation

Wasmtime traditionally manages memory using WebAssembly’s linear memory model, where each instance gets a contiguous memory block divided into 64 KiB pages. This memory can grow or shrink dynamically within defined constraints. Since Lind emulates processes as cages within a single address space, tracking allocated memory regions per cage is essential.

To address this, we integrated a vmmap system into Lind that more closely resembles POSIX-based memory management. This allows proper implementation of syscalls like brk(), mmap(), and mprotect() for memory allocation, deallocation, and permission management. It also ensures accurate memory region copying when forking cages.

### Necessary For:


#### fork()

The fork() system call requires duplicating the parent process’s memory space for the child. Properly replicating memory requires tracking protections and distinguishing shared memory regions. Some regions may have different permissions based on their initial mappings or modifications via mprotect(), preventing a simple bulk copy. Additionally, shared regions must be handled separately to maintain correct page sharing between cages. Without vmmap, accurate memory duplication in the child process wouldn’t be guaranteed.
brk()

#### brk()

The brk() system call expands the heap linearly, ensuring contiguous allocation as expected by libc and other libraries. Many functions, including malloc(), rely on this guarantee. Without a vmmap, memory allocation would follow a greedy approach, potentially interleaving heap regions with other mappings, breaking POSIX compliance, and causing library failures.

### Additional Benefits:

- Reduced fragmentation: Without memory tracking, greedy allocation wastes space by failing to reuse deallocated pages. This is particularly crucial since cages are limited to 4GB of address space.
- Improved memory safety: Heap overflows are less likely to impact valid mappings, as heaps and other memory regions remain isolated unless explicitly mapped with MAP_FIXED.##


## System Calls

### mmap()/munmap()

### brk()/sbrk()

### mprotect()