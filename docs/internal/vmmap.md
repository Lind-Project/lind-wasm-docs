# Memory Management and Vmmap

## What is a Vmmap?

A vmmap is a tool for managing the memory layout of a process within an operating system. It provides detailed information about the various memory regions allocated to the process, including the heap, stack, and memory-mapped files. The vmmap also displays the access permissions for each region, specifying whether the memory is read-only, read-write, or executable. Additionally, it highlights the start and end addresses of each region, their sizes, and any files mapped into the process's address space.

## Motivation

Wasmtime traditionally manages memory using WebAssembly's linear memory model, where each instance is allocated a contiguous block of memory divided into pages (typically 64 KiB). This memory can grow or shrink dynamically in page-sized increments, subject to limits defined by the WebAssembly module or system constraints. Because Lind emulates processes as cages within a single address space, we need to be able to track valid memory regions within each cage. We've integrated a vmmap system into Lind that manages memory more like POSIX-based systems. This addition enables proper implementation of syscalls such as brk(), mmap(), and mprotect() for allocation, deallocation, and permission management of memory.

### Necessary For:

#### fork()
The fork() system call necessitates copying the memory space of the parent process to the child. However, accurately replicating the parent's memory in the child requires tracking the memory protections of each region and identifying which memory regions are mapped as shared. Without this information, a precise representation of the parent's memory in the child cannot be ensured.

#### malloc()
Glibc's implementation of malloc relies on a POSIX style heap, which is contiguous. Using a "greedy" memory allocation style would in many cases break up the heap causing malloc() to fail.

### Additional Benefits:
- Lower fragmentation of cage memory
- Heap overflows much less likely to affect valid memory mappings, since they are positioned at opposite ends of the program unless specifically mapped with MAP_FIXED

## VMMap Implementation

### 1. Core Data Management

#### Entry Structure (VmmapEntry)
- Memory Location
  - page_num (start address)
  - npages (size)
- Permissions
  - prot (current permissions)
  - maxprot (maximum allowed)
- Backing Store
  - MemoryBackingType:
    - Anonymous
    - SharedMemory(shmid)
    - FileDescriptor(fd)
  - file_offset
  - file_size
  - cage_id

### 2. Memory Operations Flow

#### A. Memory Allocation
1. Request for Memory Space
   - find_space(npages): Searches for contiguous free pages
   - find_space_above_hint(npages, hint): Searches above specific address
   - find_map_space(num_pages, alignment): Finds aligned space
   - find_map_space_with_hint(num_pages, alignment, hint): Finds aligned space above hint

2. Memory Mapping Creation
   - add_entry(): Direct insertion with strict bounds
   - add_entry_with_overwrite(): Handles existing mappings

#### B. Memory Protection Management
- change_prot(page_num, npages, new_prot)
  1. Calculate affected region
  2. For each overlapping entry:
     - Case 1: Entry starts before region
       - Split at region start
     - Case 2: Entry extends beyond region
       - Split at region end
     - Case 3: Entry fully contained
       - Update protection
  3. Insert new intervals with updated protection

#### C. Memory Access Validation
- check_addr_mapping(page_num, npages, prot)
  1. Cache Check
     - If cached_entry covers range
     - If permissions match
  2. Live Check
     - Find overlapping entries
     - Verify permissions
     - Update cache if found
  3. Protection Enforcement
     - PROT_READ added if any protection exists

### 3. Memory Management Operations

#### A. Entry Updates
- update(page_num, npages, prot, maxprot, flags, backing, remove, ...)
  1. Validate input
     - Check npages > 0
  2. Create new entry if not removing
     - Set all entry properties
  3. Handle overlapping entries
     - insert_overwrite for atomic updates
     - Remove if needed
     - remove_overlapping for cleanup

#### B. Memory Removal
- remove_entry(page_num, npages)
  1. Create temporary mapping
  2. Overwrite existing entries
  3. Remove overlapping regions

### 4. Iterator Support

#### A. Basic Iteration
- find_page_iter(page_num)
  - Get last entry
  - Create iterator from page_num to end
- find_page_iter_mut(page_num)
  - Similar to find_page_iter
  - With mutable access

#### B. Full Map Iteration
- double_ended_iter()
  - Bidirectional iteration over all entries
- double_ended_iter_mut()
  - Mutable bidirectional iteration

### 5. Address Translation

#### WASM Support
- set_base_address(base_address)
  - Store base address for translations
- user_to_sys(address)
  - Add base_address to user address
- sys_to_user(address)
  - Subtract base_address from system address

### 6. Performance Considerations
- Cache optimization for frequent lookups
- Efficient splitting and merging of memory regions
- Minimal fragmentation through smart allocation strategies
- Fast permission checking for syscall validation

### 7. System Calls Integration

#### mmap()/munmap()

##### A. Current mmap Implementation
- Parameters:
  - addr: Suggested mapping address (can be null)
  - len: Size of mapping in bytes
  - prot: Memory protection flags (PROT_READ, PROT_WRITE, PROT_EXEC)
  - flags: Mapping flags
  - virtual_fd: Virtual file descriptor (-1 for anonymous mapping)
  - off: Offset into file

- Operation Flow:
  1. File Descriptor Handling
     - If virtual_fd != -1:
       - Translate virtual FD to kernel FD
       - Handle bad file descriptors (EBADF)
     - If virtual_fd == -1:
       - Process as anonymous mapping
  
  2. System Call
     - Direct passthrough to libc::mmap
     - Preserves all flags and protections
     - Returns actual mapped address
  
  3. Error Handling
     - EINVAL: Invalid flags or mapping failed
     - EBADF: Bad file descriptor
  
  4. Current Limitations
     - Relies on host system's mmap implementation
     - Limited validation of address ranges
     - No custom memory management

##### B. Future Improvements
- Enhanced Memory Management
  - Custom address space management
  - Validation against vmmap entries
  - Protection of Lind system memory
- Additional Features
  - MAP_FIXED handling
  - Shared memory support
  - File-backed mapping validation
  - Integration with vmmap tracking

#### brk()/sbrk()

##### A. Current Implementation
- brk syscall
  - Parameters:
    - cageid: Identifier for the calling cage
    - brk: New program break address (u32)
  - Direct passthrough to interface::brk_handler
  - Returns new break location

- sbrk syscall
  - Parameters:
    - cageid: Identifier for the calling cage
    - increment: Requested change in break location (i32)
  - Direct passthrough to interface::sbrk_handler
  - Returns previous program break
  - Negative values decrease heap size

- Key Differences:
  - brk: Sets absolute address
  - sbrk: Relative increment/decrement
  - brk uses unsigned (u32), sbrk uses signed (i32)

##### B. Handler Implementation
- brk_handler
  - Manages absolute program break location
  - Validates requested address
  - Updates vmmap entries

- sbrk_handler
  - Calculates new break from increment
  - Handles negative values (heap shrinking)
  - Maintains heap consistency

##### C. Memory Management
- Heap Growth
  - Page-aligned allocations
  - Contiguous memory space
  - Protection flags: Read/Write

- Heap Shrinking
  - Release unused pages
  - Update vmmap entries
  - Maintain minimum heap size

##### D. Error Handling
- Invalid addresses
- Out of memory conditions
- Overlapping mappings
- Protection violations

#### mprotect()

##### A. Implementation Details
- Parameters:
  - addr: Start of region to modify
  - len: Length of region
  - prot: New protection flags

- Operation Flow:
  1. Region Validation
     - Check page alignment
     - Verify address range
     - Validate protection flags
  
  2. VMMap Updates
     - Find affected entries
     - Split entries if partial overlap
     - Update protection flags
  
  3. Permission Enforcement
     - Update page table entries
     - Handle shared mappings
     - Maintain maximum permissions

- Error Handling:
  - EINVAL: Invalid addresses or protection flags
  - ENOMEM: Out of memory during operation
  - EACCES: Exceeding maximum permissions

### Common Implementation Features

#### A. Page Management
- Page Size: 64 KiB (WebAssembly default)
- Alignment: All operations page-aligned
- Protection: Per-page granularity

#### B. Error Handling
- Address validation
- Permission checking
- Resource availability

#### C. Performance Optimizations
- Caching of recent lookups
- Batch operations where possible
- Efficient range checks
