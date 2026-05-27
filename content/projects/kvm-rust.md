+++
title = 'Rust KVM Experiment'
draft = false
+++
## Overview

Virtual machine creation with KVM API in Rust.

## Technical focus

- strace, uprobes
- KVM ioctls
- Rust

## What I worked on

- tracing with strace and uprobes to find out how Qemu creates VMs
- reconstructing the Qemu logic in Rust

## Status

Experimental.

## GitHub

https://github.com/StjepanPoljak/kvm-rust
