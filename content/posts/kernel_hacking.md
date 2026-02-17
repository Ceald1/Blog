+++
title = "Kernel_hacking"
date = "2026-02-17T00:05:22-07:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "Ceald"
# authorTwitter = "" # do not include @
cover = "https://media1.tenor.com/m/oZJ_94Ngnl8AAAAd/penguin-linux.gif"
tags = ["linux", "ebpf", "lkms", "malware", "C", "programming"]
keywords = ["linux", "kernel", "hacking", "C"]
description = "A dive into linux kernel hacking"
showFullContent = false
readingTime = true
hideComments = false
+++


# Generic Title

Let's skip introductions, everyone knows what Linux is and that it's a kernel and not an OS. Most people tend to overlook how the kernel actually works not just how to install or compile packages. This article will go over how the kernel actually functions and not a "zero to hero" post.

# Building Wootkits

It's the big 26 as of writing this and writing a hooked library seems a bit outdated and generic so why not live a bit on the edge and do something directly at the kernel level?

## Limitations

Unless you're like 80 years old or something you can't modify the system call table directly anymore for safety but that's okay it's still very nice to make scary kernel modules or eBPFs.

## Hooks

In Linux you can hook system calls meaning you can intercept and modify them. If you want to see how this is done it's super easy to do using kprobes. You make a generic Linux kernel module and then create a variable with the structure type `kprobe`, the variables in the structure tell kprobes where the hook sits in the execution flow, the symbol like `"__x64_sys_kill"`. Having something like `pre_handler` set to a function means that your function will sit right after the system call is made before anything else runs. You can register probes with the `register_kprobe(&my_kprobe_structure)` function and a pointer to your kprobes structure.

An issue with using kprobes is you can't really interrupt the execution flow that much but mostly debug. Using ftrace accomplishes full system call hooking. Think of kprobes as read only mostly while ftrace is full read and write.

Ftrace allows you to fully control the system call hook as much as possible but the issue is you're manipulating the system call or overwriting the system call and can cause issues if you don't have the proper hook set up.

### eBPFs

A "safe" way of hooking into system calls is by using an eBPF but you're only given read only access to kernel memory and can't change any of it. Now this isn't super bad and totally workable but it'd only be good if you just want to do something like capture PAM credentials or lock down an entire system by denying everything access to every file. These are much harder to screw up but their ecosystem isn't as mature and more DIY than kernel modules. eBPFs are great but not there yet and still very neat for a niche skill for kernel programming.

BPFs are used for things like policy enforcement with projects like Kube Armor by hooking Linux Security Modules or LSMs. If a rootkit is an eBPF there almost certainly needs to be another program in the background that's sending out the data it collects because eBPFs are reactionary because they use system call hooks. BPFs have even more restrictions than just having read only access to kernel land by having memory limitations in place because they're ran in a VM inside of the kernel.
