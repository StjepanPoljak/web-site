+++
title = 'Rupic: Embedded Tetris Emulator'
draft = false
+++

## Overview

A simple instruction and memory emulator of PIC16F876 written in Rust with
device pipeline infrastructure. Designed specifically to emulate my old
[Embedded Tetris](../tetris-console) project. Supports custom display logic
for ST790 and keyboard input.

## Technical focus

- PIC16F876 MCU and memory internals
- Rust, PIC assembly

## What I worked on

- machine code execution loop
- guest memory implementation
- infrastructure for device pipeline
- custom keypad device and ST790 display emulation via terminal

## Status

Experimental.

## GitHub

https://github.com/StjepanPoljak/rupic
