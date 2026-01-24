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

First windows now it's time for the penguin! Unlike windows, Linux has a more direct approach for doing things like manipulating memory or linking libraries or you don't need to call 3+ api functions to do something like process injection.

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
