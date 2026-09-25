+++
title = 'Notes on x86 PVH boot'
date = 2026-09-25T16:26:03+02:00
draft = false
+++

<p align="center">
  <img src="/images/notes-on-pvh-boot-vcpu.drawio.svg" alt="PVH Boot">
</p>

## Motivation

My Rust KVM project now being able to boot Linux on ARM64, I decided to try and
boot Linux on x86. I have been trying in the past to set up the boot procedure
via `boot_params` struct. At the time it was a lot more complex and now I wanted
to have some result as soon as possible. So I turned to the PVH
(Para-Virtualized Hardware) Linux boot which simplifies this process a lot. As
a Xen Project protocol, PVH provides a relatively simple interface between a
VMM and the Linux kernel, without requiring the VMM to reproduce the
traditional firmware/bootloader path.

## Entering protected mode via BIOS

The traditional way of booting an operating system on an x86 machine is to start
in real mode and then proceed to protected mode. From there, if CPU is 64-bit,
we can go to long mode. In the PVH Linux entry path, Linux performs the
transition from the required 32-bit protected-mode entry state into 64-bit mode
itself.

### Setting up real mode

 * x86 starts at reset vector (physical address `0xFFFFFFF0`)
 * CPU is in 16-bit real mode and firmware like SeaBIOS takes over
 * SeaBIOS looks at first sector (512 bytes) of a bootable device
 * if the sector contains `0xAA55` at 510-byte offset, it's loaded at `0x7C00`
 * load the rest of data manually from our boot device (e.g. on `0x8000`)
 * jump to the code section we just loaded (e.g. to address `0x8000`)
 * if necessary, set up Stack Pointer and Stack Segment
 
### Getting to protected mode

 * make sure interrupts are disabled
 * set up and load a minimal GDT (Global Descriptor Table)
 * set PE (Protected Mode Enable) bit in CR0 (Control Register)
 * perform far jump selecting a proper GDT entry

### GDT example

A minimal GDT code set up looks like the following:

```asm
gdt_start:
	dq 0
gdt_code:			; index 1
	dw 0xFFFF		; segment limit (low)
	dw 0x0000		; segment base (low)
	db 0x00			; base address (middle)
	db 10011010b	; access bits (read/write, executable, code or data, present)
	db 11001111b	; flags (32-bit protected, 4k blocks) - segment limit (high)
	db 0x00			; base address (high)

gdt_data:			; index 2
	dw 0xFFFF		; segment limit (low)
	dw 0x0000		; segment base (low)
	db 0x00			; base address (middle)
	db 10010010b	; access bits (read/write, code or data, present)
	db 11001111b	; flags (32-bit protected, 4k blocks) - segment limit (high)
	db 0x00			; base address (high)
gdt_end:

gdt_descriptor:
	dw gdt_end - gdt_start - 1
	dd gdt_start
```

Then we enter protected mode by executing:

```asm
	cli
	lgdt [gdt_descriptor]

	mov eax, cr0
	or eax, 1
	mov cr0, eax

	jmp 0x08:protected_mode_start ; 0x08 is 1 << 3 (select index 1 in GDT)
```

The far jump reloads `CS` using the descriptor selected by `0x08`. The other
segment registers must be loaded with appropriate protected-mode selectors as
well. Each segment register has a visible selector and hidden descriptor-cache
state containing information such as the base, limit and access rights.

## How PVH helps

If we are in a virtual machine, we don't really need to go through all of these
steps. We don't need to have firmware or BIOS to set up CPU state properly, we
can simply do this from the VM.

Here, we can manipulate the vCPU to our will, and therefore we can set segment
registers ourselves to valid values, skipping GDT setup. We do need to do some
administration to help Linux understand what's going on and this is exactly
where PVH comes in.

### ELF PVH note

To determine whether an x86 Linux ELF image supports the PVH boot ABI, we can
inspect its Xen ELF notes and look for `XEN_ELFNOTE_PHYS32_ENTRY` (note type
`18`). Its descriptor gives the physical 32-bit entry point at which the guest
should be started.

Newer kernels may also contain `XEN_ELFNOTE_PHYS32_RELOC` (type `19`), which
describes alignment and physical-address constraints for relocatable PVH
kernels.

## PVH boot memory setup

Boot procedure from PVH is more straightforward, but requires more memory
management from VM:

 * get RIP from ELF PVH note entry
 * load Linux kernel ELF segments into memory
 * copy kernel command line to memory
 * construct an E820-like memory map using `HvmMemEntry` entries
 * construct and fill out `HvmStartInfo` struct
 * load memory entries and `HvmStartInfo` to memory

If using `initramfs` you also need to load it into memory, along with its
separate `initramfs` command line. We also need to fill out `HvmModuleEntry`
struct to point to `initramfs` and its command line address. Note that
here we are not talking about kernel modules, but rather Xen modules, and
for Linux that only means `initramfs`.

<p align="center">
  <img src="/images/notes-on-pvh-boot-elf.drawio.svg" alt="PVH Boot">
</p>

A good reference for the structures above is actually in the Linux kernel
headers and can be found on:

https://elixir.bootlin.com/linux/v7.1.3/source/include/xen/interface/hvm/start_info.h

## Setting up registers

 * RIP points to entrypoint from ELF note `18` entry
 * RBX points to `HvmStartInfo` physical address (this is PVH specific)
 * set up segment registers to valid Protected Mode state
 * enable PE bit in CR0 (as in ordinary boot)
 * initialize CR4 (Control Register 4) to zero
 * clear VM (bit 17), IF (bit 9) and TF (bit 8) in RFLAGS

Linux kernel-side PVH register setup is described here:

https://elixir.bootlin.com/linux/v7.1.3/source/arch/x86/platform/pvh/head.S#L30

### Segment registers

I think the best source for how to set up segment registers is actually my own
code from my Rust KVM project:

```rs
let code = kvm_segment {
	base: 0,
	limit: 0xffff_ffff,
	selector: 1 << 3,
	type_: 0b1010, // 3rd bit: code/data = 1
				   // 2nd bit: conforming = 0
				   // 1st bit: readable = 1
				   // 0th bit: accessed = 0
	present: 1,
	dpl: 0, // descriptor privilege level (0: kernel .. 3: userspace)
	db: 1,  // default operand / address size (0: 16bit, 1: 32bit)
	s: 1,   // system (1: code/data segment, 0: system segment)
	l: 0,   // long mode (0: not 64bit segment, 1: 64bit segment)
	g: 1,   // granularity (0: limit in bytes, 1: limit in 4Kb blocks)
	avl: 0, // available for software (reserved for custom OS purposes)
	unusable: 0,
	padding: 0,
};

let data = kvm_segment {
	selector: 2 << 3,
	type_: 0b0010, // 3rd bit: code/data = 0
				   // 2nd bit: conforming = 0
				   // 1st bit: writable = 1
				   // 0th bit: accessed = 0
	..code };

sregs2.tr = kvm_segment {
	base: 0,
	limit: 0x67, // as per Xen project PVH protocol
	g: 0,
	..code };
```

With BIOS boot, these segment descriptors would live in a GDT in guest memory
and the CPU would obtain their hidden descriptor state by loading the
corresponding selectors. With KVM, we can bypass the GDT entirely and directly
initialize the vCPU's segment state through `KVM_SET_SREGS`.

## Conclusion

PVH turned out to be a convenient way of getting my x86 KVM implementation to
the point where it could boot a real Linux kernel without first implementing
the traditional x86 boot protocol.

There is still plenty left to implement around the x86 side of the VMM, but
being able to reach the Linux kernel through PVH gives me a much more useful
starting point than reproducing the complete BIOS boot sequence first.

The implementation is available in the x86 initialization code in my `kvm-rust`
repository:

https://github.com/StjepanPoljak/kvm-rust/blob/main/src/arch/x86_64/init.rs

I plan to continue extending functionality of the x86 side of my VMM.
