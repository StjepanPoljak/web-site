+++
title = 'Emularium: Board Emulator'
draft = false
aliases = ['/projects/rupic/']
+++

![Emularium](/images/emularium.png)

## Overview

Emularium (formerly Rupic) emulates whole boards, not just chips. A board is
described as a KiCad schematic or a small TOML file; Emularium wires up the
parts, loads the firmware into the microcontroller and runs everything in
real time. Displays are shown over VNC, keys typed in the viewer go to the
board's keypads, and any net can be traced to a VCD file.

It started as a PIC16F876 emulator for my old
[Embedded Tetris](../tetris-console) project and is growing into an arena
for experimenting with computer architectures.

## Technical focus

- PIC16F876 MCU and memory internals
- Digital board simulation: nets, buses and timing
- Rust, PIC assembly, KiCad

## What I worked on

- machine code execution loop
- guest memory implementation
- infrastructure for device pipeline
- custom keypad device and ST7920 display emulation

## Since then

- boards loaded from KiCad schematics or TOML files
- displays served over VNC, signals traced to VCD
- 23LC1024 SPI SRAM, the first step towards external memory with the PIC's
  own RAM as a cache

## Status

Experimental.

## GitHub

https://github.com/StjepanPoljak/Emularium
