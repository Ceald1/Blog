+++
title = "Bpfs"
date = "2026-02-02T00:25:35-07:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = ""
authorTwitter = "" #do not include @
cover = "https://media1.tenor.com/m/wn22V4UMowYAAAAC/flying-bee.gif"
tags = ["malware", "ebpf", "c"]
keywords = ["bpf", "malware", "c"]
description = ""
showFullContent = false
readingTime = false
hideComments = false
+++

# Background

While I was learning about making things like rootkits and linux kernel modules (post coming soon!) I stumbled upon BPFs or Berkeley Packet Filters and you can hook system calls with them and was presented as a "safer/modern alternative" to something like a linux kernel module for hooking system calls.

# WTF is it?

It's a program that runs in a virtual machine alongside the kernel to filter things like network traffic or to secure a system, an example is Kube Armor! BPFs are also event based meaning they only trigger when certain events happen like dropping a network packet. The extended version of BPFs are eBPFs! The extended version allows for things like system calls and beyond simple packet capture and network based events, instead you can act on things like openat being called or linux security modules being called!

# Offense?

A shield is a nice tool for blocking things but you can also use it to shove things! Some prime examples are the [symbiote malware](https://www.malwarebytes.com/blog/news/2022/06/stealthy-symbiote-linux-malware-is-after-financial-institutions) and the [BPFDoor malware](https://attack.mitre.org/software/S1161/), BPFDoor specifically modifies processes without ever touching the kernel directly using well BPFs and observes packets for when to trigger/activate essentially creating a rootkit without ever touching the kernel. But how does it do this without ever writing to the kernel? Well you're able to interact with the user space on the system still and have to get creative!

## Sources

* <https://www.elastic.co/security-labs/a-peek-behind-the-bpfdoor>

# All talk no code

I'm a firm believer in no proof of concept means no exploit, theory is nice and all but what about writing one? Here's where the notes begin on my research on BPFs.

# Notes

## Memory

* Memory in eBPFs are not isolated meaning that other processes can read and interact with the maps in an eBPF.
* Maps can be sent from user space processes to eBPFs.
* eBPF maps are managed by the kernel.

## Programming

* eBPFs can have multiple programs and functions baked into one eBPF.
* IPC or interprocess communication can be performed to communicate with the user space.

## functions

* `bpf_map_lookup_elem` receives messages and looks up elements in a map.
* `bpf_obj_get_info_by_fd` receives based on file descriptor, validate the size to prevent unpredictable behavior and overflows.
* `bpf_tail_call` performs no return calls into other programs.
* `bpf_map_update_elem` updates map values.

## Sources and Resources

* <https://media.defcon.org/DEF%20CON%2027/DEF%20CON%2027%20presentations/DEFCON-27-Jeff-Dileo-Evil-eBPF-In-Depth.pdf>
* <https://github.com/Esonhugh/sshd_backdoor.git>
* <https://github.com/pathtofile/bad-bpf>
* <https://blog.tofile.dev/2021/08/01/bad-bpf.html>
* <https://media.defcon.org/DEF%20CON%2033/DEF%20CON%2033%20video%20and%20slides/DEF%20CON%2033%20-%20Jailbreaking%20the%20Hivemind%20-%20Finding%20and%20Exploiting%20Kernel%20Vulnerabilities%20in%20the%20eBPF%20Subsystem%20-%20Agostino%20%27Van1sh%27%20Panico.mp4>
