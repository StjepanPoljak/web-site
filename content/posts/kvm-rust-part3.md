+++
title = 'Building a KVM Virtual Machine in Rust: Running a binary'
date = 2026-06-20T07:12:33+02:00
+++

![Memory Diagram](/images/kvm-rust-part3.png)

## Recap

This is a continuation of [Part 2](../kvm-rust-part2) of KVM in Rust series,
where we successfully set up a memory region for our guest VM. Now we will see
how to proceed further and run a very simple binary in our hypervisor.

## Loading the binary

Now, we have only set up a memory region, but we still haven't loaded any binary
or code to actually run. For this, I have written a small "Hello world" x86
assembly file: 

https://github.com/StjepanPoljak/kvm-rust/blob/kvm-part3-code/samples/hello-world.asm

Notice that the `org 0x1000` header is quite arbitrary and it's actually a good
lead of what we also need to do (set the instruction pointer). Let's compile the
assembly file with:

```sh
nasm ./samples/hello-world.asm
```

Then, we want to load our binary into the memory region at address `0x1000`:

```rs
let mut file = File::open("./samples/hello-world")?;
let mut code = Vec::new();
file.read_to_end(&mut code)?;

unsafe {
	std::ptr::copy_nonoverlapping(code.as_ptr(),
								  (mem_ptr as *mut u8).add(0x1000),
								  code.len()) };
```

Here we are using the more optimal `copy_nonoverlapping` because we can be quite
certain that the memory where `code` is stored is not overlapping with our
memory region.

## Creating a vCPU

As we saw in [Part 1](../kvm-rust-part1), we have also discovered that QEMU also
calls `KVM_SET_VCPU` to obtain a vCPU file descriptor on which it will call
`KVM_RUN`. So, all that we have to do to set up a vCPU is to take a look at the
`ioctl` and recreate it in Rust:

```rs
// 140904 ioctl(9<anon_inode:kvm-vm>, 0xae41 /* KVM_CREATE_VCPU */, 0) = 10<anon_inode:kvm-vcpu:0>
let vcpu_fd = unsafe { libc::ioctl(self.fd, KVM_CREATE_VCPU, 0usize) };
if vcpu_fd < 0 {
	return Err(io::Error::last_os_error());
}
```

### Investigating vCPU operations

It would be tempting to simply call `KVM_RUN` on this file descriptor, however,
we don't really know how vCPU is set up, where the instruction pointer is and
even if the registers are in the valid state. If we look at the `ioctl` calls
referring to `kvm-vcpu:0` in our log (and filtering out the spamming `KVM_RUN`
and `KVM_SMI`), we can see a lot of lines like these being repeated:

```text
$ grep 'kvm-vcpu:0' kvm.log | grep -v 'RUN\|SMI'

                ### omitted a lot of repetitive strace output ###

140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xc080aebe /* KVM_GET_NESTED_STATE */, 0x7768f8002010) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xc028ae92 /* KVM_TPR_ACCESS_REPORTING */, 0x77690198f090) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4140aecd /* KVM_SET_SREGS2 */, 0x77690198f2f0) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4090ae82 /* KVM_SET_REGS */, {rax=0x20, ..., rsp=0x6d88, rbp=0, ..., rip=0x18, rflags=0x246}) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x5000aea5 /* KVM_SET_XSAVE */, 0x7768f8001000) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4188aea7 /* KVM_SET_XCRS */, 0x77690198f2c0) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4008ae89 /* KVM_SET_MSRS */, 0x7768f80040a0) = 59
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4040aea0 /* KVM_SET_VCPU_EVENTS */, 0x77690198f500) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4008ae89 /* KVM_SET_MSRS */, 0x7768f80040a0) = 1
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4080aea2 /* KVM_SET_DEBUGREGS */, 0x77690198f500) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xc008ae88 /* KVM_GET_MSRS */, 0x77690198f5a0) = 1
```

So we see that QEMU is setting vCPU registers via `KVM_SET_REGS` and in this log
we can see their exact values. It's also calling `KVM_SET_SREGS2` which is
setting system registers; unfortunately, we cannot see the values used here.
Also, we do not need to follow the whole logic here, we just want to see what
QEMU is doing with registers before calling `KVM_RUN`:

```text
$ awk '/KVM_RUN/ {print; exit} {print}' kvm.log | grep '[GS]ET_[S]\?REGS\|KVM_RUN'
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4140aecd /* KVM_SET_SREGS2 */, 0x77690198f310) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4090ae82 /* KVM_SET_REGS */, {rax=0, ..., rsp=0, rbp=0, ..., rip=0xfff0, rflags=0x2}) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x8090ae81 /* KVM_GET_REGS */, {rax=0, ..., rsp=0, rbp=0, ..., rip=0xfff0, rflags=0x2}) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x8140aecc /* KVM_GET_SREGS2 */, 0x77690198f310) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4140aecd /* KVM_SET_SREGS2 */, 0x77690198f310) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4090ae82 /* KVM_SET_REGS */, {rax=0, ..., rsp=0, rbp=0, ..., rip=0xfff0, rflags=0x2}) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xae80 /* KVM_RUN */, 0) = -1 EINTR (Interrupted system call)
```

Since QEMU first calls `SET_SREGS2` and `SET_REGS` we should actually
investigate what the default (if any) values are in our own implementation.

### Implementing register get/set operations in Rust

The procedure for this is as with all other KVM `ioctl` calls we implemented.
We find out how to construct their numbers (consult the previous article here)
and reimplement these constants in Rust:

```rs
const KVM_GET_SREGS2 : u64 = _IOR::<kvm_sregs2>(KVMIO, 0xcc);
const KVM_SET_SREGS2 : u64 = _IOW::<kvm_sregs2>(KVMIO, 0xcd);
const KVM_GET_REGS : u64 = _IOR::<kvm_regs>(KVMIO, 0x81);
const KVM_SET_REGS : u64 = _IOW::<kvm_regs>(KVMIO, 0x82);
```

We also need to add `kvm_sregs2` and `kvm_regs` in our `build.rs` file:

```rs
    bindgen::Builder::default()
        .header("/usr/include/linux/kvm.h")
        .allowlist_type("kvm_sregs2")
        .allowlist_type("kvm_userspace_memory_region")
        .allowlist_type("kvm_regs")
        .generate_comments(false)
        .generate()?
        .write_to_file(out_path.join("kvm-bindings.rs"))?;
```

Then, `ioctl` calls are as follows (I will only give example for `get_sregs2`
and `get_regs`):

```rs
let mut sregs2 = unsafe { std::mem::zeroed() };
let ret = unsafe { libc::ioctl(self.fd, KVM_GET_SREGS2, &mut sregs2) };

let mut regs = unsafe { std::mem::zeroed() };
let ret = unsafe { libc::ioctl(self.fd, KVM_GET_REGS, &mut regs) };
```

### Peeking at registers

After obtaining `sregs2` and `regs` we can print them out and see what they are
set to by default (to accomplish this we consult the `struct` definition in our
`kvm-bindings.rs` (or in the Linux headers) and print out all the values of the
fields we find). For `regs` we see they are pre-set to:

```text
RAX=0x0 RBX=0x0 RCX=0x0 RDX=0x600
RSI=0x0 RDI=0x0 RSP=0x0 RBP=0x0
R8=0x0  R9=0x0  R10=0x0 R11=0x0
R12=0x0 R13=0x0 R14=0x0 R15=0x0
RIP=0xfff0      RFLAGS=0x2
```

Taking a look at our trace from `kvm.log` we can see that the ones QEMU sets are
exactly the same (except for `RDX` but we can get away with this one).

### Peeking at system registers

Now, for `sregs2` it gets a bit more complicated. Our defaults are as follows:

```text
CS      base=0xffff0000 selector=0xf000 limit=0xffff    type=0xb        present=0x1
        dpl=0x0         db=0x0          s=0x1   l=0x0   g=0x0           avl=0x0

DS      base=0x0        selector=0x0    limit=0xffff    type=0x3        present=0x1
        dpl=0x0         db=0x0          s=0x1   l=0x0   g=0x0           avl=0x0

ES      base=0x0        selector=0x0    limit=0xffff    type=0x3        present=0x1
        dpl=0x0         db=0x0          s=0x1   l=0x0   g=0x0           avl=0x0

FS      base=0x0        selector=0x0    limit=0xffff    type=0x3        present=0x1
        dpl=0x0         db=0x0          s=0x1   l=0x0   g=0x0           avl=0x0

GS      base=0x0        selector=0x0    limit=0xffff    type=0x3        present=0x1
        dpl=0x0         db=0x0          s=0x1   l=0x0   g=0x0           avl=0x0

SS      base=0x0        selector=0x0    limit=0xffff    type=0x3        present=0x1
        dpl=0x0         db=0x0          s=0x1   l=0x0   g=0x0           avl=0x0

TR      base=0x0        selector=0x0    limit=0xffff    type=0xb        present=0x1
        dpl=0x0         db=0x0          s=0x0   l=0x0   g=0x0           avl=0x0

LDT     base=0x0        selector=0x0    limit=0xffff    type=0x2        present=0x1
        dpl=0x0         db=0x0          s=0x0   l=0x0   g=0x0           avl=0x0

GDT     base=0x0        limit=0xffff

IDT     base=0x0        limit=0xffff

CR0=0x60000010          CR2=0x0         CR3=0x0         CR4=0x0         CR8=0x0

EFER=0x0                APIC_BASE=0xfee00900            FLAGS=0x0

PDPTRS[0]=0x0           PDPTRS[1]=0x0           PDPTRS[2]=0x0           PDPTRS[3]=0x0
```

Problem is that we don't see any values in our `strace` log, only the address of
the variable. So we need to be a bit creative here if we want to find out what 
QEMU sets these values to. We could use `ptrace` (in fact `strace` is built upon
`ptrace` API), but it may be a bit too much. Same for `uprobes` and `eBPF`. We
do have GDB, though, and it's just perfect as a one-off thing here. All we have
to do is run QEMU under GDB and then execute:

```text
(gdb) break ioctl if $rsi == 0x4140aecd
(gdb) run
```

Note that `0x4140aecd` is the exact value that we extracted from `SET_SREGS2`
`ioctl` call (in x86 ABI, `RSI` is holding the value of second argument):

```text
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4140aecd /* KVM_SET_SREGS2 */, 0x77690198f310) = 0
```

Once GDB breaks, we can dump memory. However, we don't know the exact size of
`struct kvm_sregs2`; we could guess or manually inspect, but quickest way to get
it is to actually have a small C program:

```c
#include <linux/kvm.h>
#include <stdlib.h>
#include <stdio.h>

int main() {
  printf("%lu", sizeof(struct kvm_sregs2));
  return 0;
}
```

The program will return value of `320` (at least on my system) and we can then
use this in GDB:

```text
(gdb) dump memory dump.bin $rdx $rdx+320
```

Note that `RDX` is holding the third argument, i.e. the address we actually saw
in our `strace` log. Now, we can reuse our function for printing `sregs2` in
Rust. We simply hack our main function to load the file and reinterpret its data
as `sregs2` and then print it. We got:

```text
CS	base=0xffff0000	selector=0xf000	limit=0xffff	type=0xb	present=0x1
	dpl=0x0		db=0x0		s=0x1	l=0x0	g=0x0		avl=0x0

DS	base=0x0	selector=0x0	limit=0xffff	type=0x3	present=0x1
	dpl=0x0		db=0x0		s=0x1	l=0x0	g=0x0		avl=0x0

ES	base=0x0	selector=0x0	limit=0xffff	type=0x3	present=0x1
	dpl=0x0		db=0x0		s=0x1	l=0x0	g=0x0		avl=0x0

FS	base=0x0	selector=0x0	limit=0xffff	type=0x3	present=0x1
	dpl=0x0		db=0x0		s=0x1	l=0x0	g=0x0		avl=0x0

GS	base=0x0	selector=0x0	limit=0xffff	type=0x3	present=0x1
	dpl=0x0		db=0x0		s=0x1	l=0x0	g=0x0		avl=0x0

SS	base=0x0	selector=0x0	limit=0xffff	type=0x3	present=0x1
	dpl=0x0		db=0x0		s=0x1	l=0x0	g=0x0		avl=0x0

TR	base=0x0	selector=0x0	limit=0xffff	type=0xb	present=0x1
	dpl=0x0		db=0x0		s=0x0	l=0x0	g=0x0		avl=0x0

LDT	base=0x0	selector=0x0	limit=0xffff	type=0x2	present=0x1
	dpl=0x0		db=0x0		s=0x0	l=0x0	g=0x0		avl=0x0

GDT	base=0x0	limit=0xffff

IDT	base=0x0	limit=0xffff

CR0=0x60000010		CR2=0x0		CR3=0x0		CR4=0x0		CR8=0x0

EFER=0x0		APIC_BASE=0xfee00900		FLAGS=0x0

PDPTRS[0]=0x0		PDPTRS[1]=0x4b275f5fce32f200		PDPTRS[2]=0x0		PDPTRS[3]=0x3
```

We can see that this more or less corresponds to the defaults we observed in our
KVM implementation. If we want to change our registers (which we will), then we
will always first get the ones that are current in the vCPU, change the ones we
need and then set them using `KVM_SET_REGS` (or `KVM_SET_SREGS2`). So one of
first things to do is to set `CS` to flat mapping:

```rs
let mut sregs2 = vcpu.get_sregs2()?;
sregs2.cs.base = 0;
sregs2.cs.selector = 0;
vcpu.set_sregs2(sregs2)?;
```

Recall that our assembly file was assembled with `org 0x1000`, so setting RIP to
`0x1000` causes execution to begin at the start of the loaded binary:

```rs
let mut regs = vcpu.get_regs()?;
regs.rip = 0x1000;
vcpu.set_regs(regs)?;
```

Furthermore, we now have a very good process of finding out what QEMU does with
registers.

## Running the binary

Now this is where `strace` stopped being useful and my understanding of the KVM
run loop became more useful. Another positive thing is that this part of QEMU
source code is quite readable and can be found in function [kvm_vcpu_thread_fn](https://elixir.bootlin.com/qemu/v11.0.1/source/accel/kvm/kvm-accel-ops.c).

The more x86-specific code with exit reasons can be found in the [kvm_arch_handle_exit](https://elixir.bootlin.com/qemu/v11.0.1/source/target/i386/kvm/kvm.c) function.

### Setting kvm_run shared memory region

To wrap things up, first we need to mmap the shared kvm_run structure that KVM
uses to communicate VM-exit information and other runtime state between the
kernel and userspace:

```rs
// 140904 ioctl(3</dev/kvm<char 10:232>>, 0xae04 /* KVM_GET_VCPU_MMAP_SIZE */, 0) = 12288
let kvm_run_size = unsafe {
	libc::ioctl(kvm_fd, KVM_GET_VCPU_MMAP_SIZE, 0usize) };
let kvm_run_mem = unsafe {
	libc::mmap(ptr::null_mut(), kvm_run_size as usize,
			   libc::PROT_READ|libc::PROT_WRITE, libc::MAP_SHARED,
			   vcpu_fd, 0) };
```

Then, when we call `KVM_RUN` we simply match the exit reasons and act
accordingly:

```rs
loop {
	unsafe { libc::ioctl(vcpu_fd, KVM_RUN, 0usize) };
	let run = unsafe { *(kvm_run_mem as *mut kvm_run) };

	match run.exit_reason {
		KVM_EXIT_MMIO => { /* omitted */ }
		KVM_EXIT_HLT => {
			println!("Guest halted.");
			break; }
		KVM_EXIT_SHUTDOWN => {
			println!("Guest shutdown.");
			break; }
		KVM_EXIT_INTERNAL_ERROR => {
			return Err(io::Error::other("KVM internal error.")); }
		_ => {
			println!("EXIT REASON = {}", run.exit_reason);
		}
	}
}
```

### Our first output

Being that our assembly file uses VGA MMIO to output the "Hello world!" string,
I have only implemented `KVM_EXIT_MMIO`. Then we can finally see the output:

```text
$ cargo run ./samples/hello-world
     ## ommitted sregs2 output and warnings ##
RAX=0x0 RBX=0x0 RCX=0x0 RDX=0x600
RSI=0x0 RDI=0x0 RSP=0x0 RBP=0x0
R8=0x0  R9=0x0  R10=0x0 R11=0x0
R12=0x0 R13=0x0 R14=0x0 R15=0x0
RIP=0x1000      RFLAGS=0x2
Hello from KVM!
Guest halted.
RAX=0xb800      RBX=0x0 RCX=0x0 RDX=0x600
RSI=0x102d      RDI=0x22        RSP=0x0 RBP=0x0
R8=0x0  R9=0x0  R10=0x0 R11=0x0
R12=0x0 R13=0x0 R14=0x0 R15=0x0
RIP=0x101b      RFLAGS=0x46
```

## Conclusion

This concludes the three-part series on figuring out KVM via `strace` and
reimplementing it in Rust. A lot of next steps actually come down to x86
architecture and bootloader specifics, so we may leave it here for now. The full
code capable of running simple binaries can be found on GitHub:

https://github.com/StjepanPoljak/kvm-rust/tree/kvm-part3-code
