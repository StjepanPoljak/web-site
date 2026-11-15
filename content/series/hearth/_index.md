+++
title = 'Hearth'
summary = 'Building a KVM virtual machine monitor in Rust, from the first strace of QEMU to Linux on three architectures and Doom.'
project = '/projects/hearth/'
featureimage = 'images/hearth.png'
arcs = [
  { title = 'Learning KVM on x86', from = 1, to = 3, desc = 'Reverse-engineering QEMU with strace and rebuilding the essentials in Rust.' },
  { title = 'Porting to ARM64', from = 4, to = 5, desc = 'The same hypervisor on a Raspberry Pi 4.' },
  { title = 'Booting Linux on ARM64', from = 6, to = 7 },
  { title = 'Linux on x86: PVH and Virtio', from = 8, to = 9, desc = 'Implementation notes for the x86 boot path and the first virtio device.' },
  { title = 'Doom and paging', from = 10, to = 12, desc = 'Hearth gets its name, a VGA display, and a closer look at how KVM handles guest memory.' },
]
+++

[Hearth](/projects/hearth/) is a small virtual machine monitor for Linux KVM,
written in Rust from scratch. It boots Linux on x86_64, aarch64 and riscv64,
and on x86 it also runs bare-metal Doom.

It started as an experiment in creating virtual machines with the KVM API in
Rust, and the early parts of this series still call it "my Rust KVM
hypervisor". Read in order, the series follows that experiment growing into a
VMM.
