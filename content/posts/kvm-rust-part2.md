+++
title = 'Building a KVM Virtual Machine in Rust: Memory Setup'
date = 2026-06-09T17:30:10+02:00
+++

## Recap

This is a continuation of my [previous article](../kvm-rust-part1) which dealt
with reverse-engineering QEMU with `strace` to learn how KVM works. Now it's
time to try and follow the steps we got from the `strace` logs to build our own
KVM-based virtual machine in Rust.

## KVM Headers

I haven't actually used existing KVM libraries written specifically for Rust but
opted to use the `libc` crate which provides the required `ioctl` bindings and
helper macros. The main reason is that I want a complete understanding of what
is happening under the hood. Now for each of these KVM `ioctl` calls we can use
the Linux headers for reference. For example, to find out how to construct
`ioctl` number for `KVM_CREATE_VM` we simply can do:

```text
$ grep -Rn 'KVM_CREATE_VM' /usr/include/linux/
/usr/include/linux/kvm.h:855:/* machine type bits, to be used as argument to KVM_CREATE_VM */
/usr/include/linux/kvm.h:882:#define KVM_CREATE_VM             _IO(KVMIO,   0x01) /* returns a VM fd */
```

Luckily in `libc` Rust crate we have macros for `_IO` (and the like), but we
still need `KVMIO` macro:

```text
$ grep -Rn 'define KVMIO' /usr/include/linux/
/usr/include/linux/kvm.h:853:#define KVMIO 0xAE
```

We can now construct the `ioctl` number for `KVM_CREATE_VM`:

```rs
use libc::{_IOW, _IO, _IOR};

const KVMIO : u32 = 0xae;
const KVM_CREATE_VM : u64 = _IO(KVMIO, 0x01);
```

Note that exact integer type depends on the platform and `libc` definitions.
Then, in the main we can do:

```rs
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .open("/dev/kvm")
        .expect("failed to open /dev/kvm");

    let fd = file.as_raw_fd();
    let vm_fd = unsafe { libc::ioctl(fd, KVM_CREATE_VM, 0usize) };
	Ok(())
}
```

This is the procedure we will follow for each relevant `ioctl` call. In fact,
we can "reverse-engineer" our own program and then compare it with the original,
to make sure we are doing the right thing:

```text
$ strace cargo run

     ### omitted irrelevant strace output ###

openat(AT_FDCWD, "/dev/kvm", O_RDWR|O_CLOEXEC) = 3
ioctl(3, KVM_CREATE_VM, 0)              = 4
```
### Important note

In production code we would immediately check for a negative return value and
convert errno into a Rust error. To keep the example focused, I am omitting
proper error handling in this article.

## Setting memory region

Now, to recall, the next step is setting up the memory region used by both the
KVM guest and the host. This region will be the memory of our virtual machine, a
place where we will load our binary:

```text
140900 mmap(NULL, 1075838976, 0 /* PROT_NONE */, 0x22 /* MAP_PRIVATE|MAP_ANONYMOUS */, -1, 0) = 0x7768b3e00000
140900 mmap(0x7768b3e00000, 1073741824, 0x3 /* PROT_READ|PROT_WRITE */, 0x32 /* MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS */, -1, 0) = 0x7768b3e00000
140900 ioctl(9<anon_inode:kvm-vm>, 0x4020ae46 /* KVM_SET_USER_MEMORY_REGION */, {slot=0, flags=0, guest_phys_addr=0, memory_size=1073741824, userspace_addr=0x7768b3e00000}) = 0
```

### Recreating mmap call

So, first thing we need to do is follow the same logic for `mmap` which is also
available in the `libc` Rust crate. After creating the virtual machine, we could
simply recreate our own `mmap` calls based on the `strace` output. However,
notice that QEMU first reserves a larger address range with `PROT_NONE` and then
maps only the portion it actually intends to use. For our prototype we do not
actually need to mimic this exact reservation pattern.

```rs
let mem_size: u64 = 256 * 1024;
let mem = unsafe {
	libc::mmap(ptr::null_mut(),
			   mem_size as usize,
			   libc::PROT_READ|libc::PROT_WRITE,
			   libc::MAP_PRIVATE|libc::MAP_ANONYMOUS,
			   -1,
			   0)
};
```

For this experiment we are only allocating 256 kilobytes because we are not yet
booting a full operating system and therefore need very little guest memory.
Also note that, as with `ioctl`, production code should check whether mmap()
returned `MAP_FAILED`.

### Recreating ioctl call

First we need to see the definition of the `KVM_SET_USER_MEMORY_REGION`:

```text
$ grep -Rn 'define KVM_SET_USER' /usr/include/linux/ -A1
/usr/include/linux/kvm.h:1433:#define KVM_SET_USER_MEMORY_REGION _IOW(KVMIO, 0x46, \
/usr/include/linux/kvm.h-1434-                                  struct kvm_userspace_memory_region)
```

We see that for this one, we need `struct kvm_userspace_memory_region`. The
values we need are visible in the `strace` output, but copying struct definition
to our Rust program is really not advisable. Luckily, Rust has `bindgen` which
we can use to get this KVM struct (and others) from Linux headers. For this
purpose we have a separate `build.rs` file which will contain:

```rs
use bindgen;
use std::path::PathBuf;
use std::env;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let out_path = PathBuf::from(env::var("OUT_DIR")?);

    bindgen::Builder::default()
		.header("/usr/include/linux/kvm.h")
		.allowlist_type("kvm_userspace_memory_region")
		.generate_comments(false)
		.generate()?
		.write_to_file(out_path.join("kvm-bindings.rs"))?;

    Ok(())
}
```


Now in the `main.rs` we can import these bindings with:

```rs
include!(concat!(env!("OUT_DIR"), "/kvm-bindings.rs"));
```

Then, after `mmap()` calls, we set the memory region, also imitating what
QEMU is doing in the `strace` output:

```rs
let region = kvm_userspace_memory_region {
	slot : 0,
	flags : 0,
	guest_phys_addr : 0x0,
	memory_size : mem_size,
	userspace_addr : mem as u64
};

let _ret = unsafe { libc::ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region) };
```

We register this region starting at guest physical address `0x0`, meaning the
first byte of our allocated host memory will appear as physical address 0 inside
the guest. Also note that the `KVM_SET_USER_MEMORY_REGION` call does not copy
memory, but rather tells KVM that guest physical address will be backed by a
specific userspace memory region.

## Running the code

Next thing to do is to run it. We still haven't added any checks after `mmap`
and `ioctl` calls, but for this prototype we can again simply use `strace` our
own code:

```text
$ strace -yy -X verbose -e trace=ioctl,mmap,openat,read,write cargo run

                ### omitted irrelevant strace output ###

openat(-100 /* AT_FDCWD */</home/stjepan/Develop/KVM/rust>, "/dev/kvm", 0x80002 /* O_RDWR|O_CLOEXEC */) = 3</dev/kvm<char 10:232>>
ioctl(3</dev/kvm<char 10:232>>, 0xae01 /* KVM_CREATE_VM */, 0) = 4<anon_inode:kvm-vm>
mmap(NULL, 262144, 0x3 /* PROT_READ|PROT_WRITE */, 0x22 /* MAP_PRIVATE|MAP_ANONYMOUS */, -1, 0) = 0x7d63fdfa2000
ioctl(4<anon_inode:kvm-vm>, 0x4020ae46 /* KVM_SET_USER_MEMORY_REGION */, {slot=0, flags=0, guest_phys_addr=0, memory_size=262144, userspace_addr=0x7d63fdfa2000}) = 0
```

We can see our output is fine and no errors were reported. Note that a full
working example with proper checking and Rust idiomatic approaches can be found
on my GitHub page:

https://github.com/StjepanPoljak/kvm-rust/tree/kvm-part1-code

### Next steps

At this point we have a VM object and guest memory, but nothing is actually executing yet. In the next part we will create a vCPU, initialize its state, load a small binary into guest memory and enter the first `KVM_RUN` loop.
