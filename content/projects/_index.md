---
title: "Projects"
summary: "Hypervisors, emulators and the low-level experiments that led to them."
_build:
  list: never
cascade:
  showDate: false
  showReadingTime: false
  showPagination: false
  showRelatedContent: false
  showTableOfContents: false
  showTaxonomies: false
---

<h2 class="project-section-title">Current</h2>

<div class="project-featured">

{{< project-feature
  title="Emularium"
  desc="Board emulator that runs whole boards from KiCad schematics: PIC16F876, LCD, keypad and SPI RAM. It even runs a small language model."
  link="/projects/emularium/"
  image="/images/emularium.png"
  series="Emularium"
  github="https://github.com/StjepanPoljak/Emularium"
>}}

{{< project-feature
  title="Hearth"
  desc="Virtual machine monitor in Rust on KVM. Boots Linux on x86_64, aarch64 and riscv64, and runs Doom."
  link="/projects/hearth/"
  image="/images/hearth.png"
  series="Hearth"
  github="https://github.com/StjepanPoljak/Hearth"
>}}

</div>

<h2 class="project-section-title">Earlier projects</h2>

<div class="project-grid">

{{< project-card
  title="Embedded Tetris (PIC Assembly)"
  desc="Two-PIC system with IR remote input and custom game logic. Where Emularium started."
  link="/projects/tetris-console/"
  meta="PIC assembly · hardware"
>}}

{{< project-card
  title="Experimental Operating System"
  desc="Boot process, kernel structure and low-level system design on ARM64 and x86."
  link="/projects/raspios/"
  meta="Assembly · C"
>}}

{{< project-card
  title="Melodeer"
  desc="Low-level audio pipeline in C for playing FLAC files including TUI."
  link="/projects/melodeer/"
  meta="C · OpenAL"
>}}

{{< project-card
  title="Melodeer Remote"
  desc="Android client and Python server for controlling audio on Raspberry Pi."
  link="/projects/melodeer-remote/"
  meta="Android · Python"
>}}

{{< project-card
  title="X Server Exploration (Haskell)"
  desc="Frontend on top of X Server written in Haskell."
  link="/projects/xmenu-hs/"
  meta="Haskell · X11"
>}}

</div>
