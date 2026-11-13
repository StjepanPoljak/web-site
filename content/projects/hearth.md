+++
title = 'Hearth: KVM Virtual Machine Monitor'
draft = false
aliases = ['/projects/kvm-rust/']
+++

![Hearth](/images/hearth.png)

## Overview

Hearth is a small virtual machine monitor for Linux KVM, written in Rust
from scratch. It boots Linux on x86_64, aarch64 and riscv64, and on x86 it
also runs bare-metal Doom. Guests get a serial console, virtio-blk and
virtio-net devices, and on x86 a VGA display and PS/2 keyboard served over
VNC.

It started as an experiment in creating virtual machines with the KVM API
in Rust.

## Technical focus

- strace, uprobes, bpftrace
- KVM ioctls, paging and virtio
- Rust

## What I worked on

- tracing with strace and uprobes to find out how Qemu creates VMs
- reconstructing the Qemu logic in Rust

## Since then

- Linux boot on x86_64 (PVH), aarch64 and riscv64
- virtio-blk and virtio-net over MMIO, 16550 UART, i8042 keyboard
- VGA mode 13h over VNC, enough to run Doom as a bare-metal guest
- shadow paging experiments, traced with BPF

## Status

Experimental.

## GitHub

https://github.com/StjepanPoljak/Hearth
