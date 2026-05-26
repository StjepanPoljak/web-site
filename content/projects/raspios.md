+++
title = 'Experimental Operating System'
draft = false
+++

## Overview

An exploration in operating system development, started on a Raspberry Pi 3B+,
but lately also expanded to x86. Boot stages written in ARM64 and x86 assembly,
with later memory allocator and higher-level logic written in C. Build system
written in CMake, including Linux kernel-like configuration. Support tooling
written in Python and ready-to-use Qemu setup for testing both ARM64 and x86
versions.

## Technical focus

- operating system development
- assembly programming (ARM64 and x86)
- C, CMake
- linker scripts, tooling in Python

## What I worked on

- ARM64 boot-up process up to and including basic MMU setup
- first two boot stages in x86 up to fetching BIOS memory map
- basic memory allocation
- basic interrupt and serial device support

## Status

Experimental.

## GitHub

https://github.com/StjepanPoljak/raspios
