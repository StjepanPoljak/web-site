---
title: "Projects"
_build:
  list: never
---

## Low-level Systems

<div class="project-grid">

{{< project-card
  title="Embedded Tetris (PIC Assembly)"
  desc="Two-PIC system with IR remote input and custom game logic."
  link="/projects/tetris-console/"
>}}

{{< project-card
  title="Melodeer"
  desc="Low-level audio pipeline in C for playing FLAC files including TUI."
  link="/projects/melodeer/"
>}}

{{< project-card
  title="Experimental Operating System"
  desc="Boot process, kernel structure and low-level system design."
  link="/projects/raspios/"
>}}

</div>


## Virtualization & Emulation

<div class="project-grid">

{{< project-card
  title="Hearth: KVM Virtual Machine Monitor"
  desc="VMM in Rust on KVM. Boots Linux on x86_64, aarch64 and riscv64, and runs Doom."
  link="/projects/hearth/"
>}}

{{< project-card
  title="Emularium: Board Emulator"
  desc="Runs whole boards from KiCad schematics: PIC16F876, LCD, keypad and SPI RAM."
  link="/projects/emularium/"
>}}

</div>



## Systems & Infrastructure

<div class="project-grid">

{{< project-card
  title="Melodeer Remote"
  desc="Android client and Python server for controlling audio on Raspberry Pi."
  link="/projects/melodeer-remote/"
>}}

{{< project-card
  title="X Server Exploration (Haskell)"
  desc="Frontend on top of X Server written in Haskell."
  link="/projects/xmenu-hs/"
>}}

</div>
