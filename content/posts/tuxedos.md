+++
title = "Tuxedos"
date = "2026-01-24T00:08:40-07:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = ""
authorTwitter = "" #do not include @
cover = "https://media.tenor.com/bhewUhwCTYYAAAAi/tux-linux-tux.gif"
tags = ["malware", "research", "linux", "zig"]
keywords = ["", ""]
description = "A dive into the linux kernel on system calls and malware development research (POC code coming soon 👀)."
showFullContent = false
readingTime = true
hideComments = false
+++

# Background

First windows now it's time for the penguin! Unlike windows, Linux has a more direct approach for doing things like manipulating memory or linking libraries or you don't need to call 3+ API functions to do something like process injection.

# Fundamentals

## Memory

Everyone knows things like RAM but what about the other memory in the system?

### Primary

* Your RAM and ROM (read only memory)
* Disposable/wiped after a reboot.
* Fast readily available for programs

#### ROM

* Read only.
* BIOS level.
* Smaller chip.
* Provides all boot instructions and firmware.

----------------------

#### Register memory

* storage locations on the CPU that store temporary data.
* Faster than RAM.
* Important instructions
* each register typically holds between 32 and 64 bits of data.
* THIS IS NOT CACHE MEMORY!!

##### Data Register

* 16 bit register that holds variables.
* Temporary holding place for data.

##### Program Counter Register (PC Register)

* Memory address for next set of instructions in the program.
* Keeps proper sequence in the program.

##### Instruction Register

* 16 bit register that contains the current instruction code from the main memory (RAM).
* This is what the CPU actually executes.

##### Address Register

* 12 bit register for address location.
* CPU fetches and handles instructions from this.

##### I/O Address Register

* unique address with an input or output device like a keyboard or audio.
* CPU uses this to interact with other devices.

##### I/O Buffer Register

* temporary buffer for the I/O Address Register to exchange and hold data.
* Deals with before and after processing.

----------------------

#### Cache Memory

* Fast but small.
* Typically old memory.
* CPU checks the cache first (a cache hit) before reading the RAM (a cache miss if not in cache).

##### L1 (Level 1 Cache)

* First level in the CPU.
* Ranges from 2KB to 64KB in size.
* Every core has this.

##### L2 (Level 2 Cache)

* Might not be present in the CPU.
* 2 cores may share it.
* 256KB to 512KB in size.

##### L3 (Level 3 Cache)

* Shared by all cores and present outside of the CPU.
* Ranges from 1MB to 8MB in size.

----------------------

### Secondary

* Optical Disks
* Flash memory
* Slower than primary
* Persistent

### Sources

* <https://www.geeksforgeeks.org/computer-organization-architecture/introduction-to-memory-and-memory-units/>
* <https://www.geeksforgeeks.org/computer-organization-architecture/memory-hierarchy-design-and-its-characteristics/>
* <https://www.geeksforgeeks.org/computer-science-fundamentals/cache-memory/>

## Assembly

Just kidding lol, I won't have notes on assembly, resources bellow already did it.

### Sources

* <https://c9x.me/articles/gthreads/mach.html>
* <https://cs4157.github.io/www/2024-1/lect/13-x86-assembly.html>

## Memory Management in Linux

### Virtual Memory

* A technique that allows for Linux to use more memory than physically available.
* Uses disk storage as an extension.
* Allows for multitasking and memory isolation between processes.

#### Virtual address space

* text segment: contains actual code for a program.
* Data segment: stores variables.
* Heap: dynamically allocated memory region that can grow.
* Stack: all function call frames, local variables, and control flow data.
The flow of the virtual address space goes downwards meaning the stack interacts with the heap then data, then code.

### Sources

* <https://infosecbytes.io/linux-internals-a-deep-dive-into-memory-management/>

# Process Injection

## Background and important

In Linux there's no possible way to allocate more memory to a process meaning if the original process isn't restored it'll crash.

## System Calls and Methods

* `ptrace`: debugs a remote process meaning memory on that process can be changed and inspected.
* `procfs`: a filesystem that shows the interfaces for running processes (literally in `/proc` on Linux).
  * Processes are typically directories represented by their PIDs.
  * Inside of the PIDs is the mem file that shows the memory address and space for that process.
* `process_vm_writev`: allows for modifying data space of the remote process.
  * This syscall receives a pointer and copies it to the specified location in the remote process.

### The Elephant In The Room

All this is neat and all but how do things like the `ps` command get all of the processes and memory in Linux? Well it actually reads them from the `/proc` directory on the Linux filesystem!

## Sources

* <https://www.akamai.com/blog/security-research/the-definitive-guide-to-linux-process-injection>
* <https://stackoverflow.com/questions/1401359/understanding-linux-proc-pid-maps-or-proc-self-maps>
* <https://unix.stackexchange.com/questions/262177/how-does-the-ps-command-work>

# Function Hooking

## Background and important

In Linux you can redirect, intercept and alter function calls at run time for an app like setting a fixed number instead of a random one when calling rand from libc. Bellow is a digram from infosecwriteups.com on how function hooking works.
![diagram](https://miro.medium.com/v2/resize:fit:640/format:webp/1*iBk3WT2bqoHKaPcL0CsAkg.png)
(source of image is in sources)

## Methods

* `LD_PRELOAD` environment variable: this variable is loading libraries before executing a program.

## Sources

* <https://infosecwriteups.com/a-gentle-introduction-to-function-hooking-using-ld-preload-1714124a6eb9>
