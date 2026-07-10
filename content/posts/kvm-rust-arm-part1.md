+++
title = 'Porting my Rust KVM hypervisor to ARM64: Working with registers'
date = 2026-07-03T19:26:33+02:00
+++

![Registers and error](/images/kvm-rust-arm-part1.png)

## Introduction

This is a follow-up to my KVM in Rust series where I showed how to use `strace`
and `GDB` to reverse-engineer QEMU/KVM and reimplement pieces of it in my own
hypervisor in Rust. So far this was done on a modern x86 CPU. This time we'll
see how to get KVM running on 64-bit ARM architecture. For this I have used my
Raspberry Pi 4B which has a CPU with hardware virtualization support.

## Cross-compilation

The first thing to do was to set up an environment for cross-compilation and for
this kind of thing I almost always use docker. So I have written a simple
`Dockerfile` installing the cross-compilation toolchain for 64-bit ARM along
with Rust essentials:

https://github.com/StjepanPoljak/kvm-rust/blob/kvm-arm-code/Dockerfile

Building this image is straightforward, but it does require passing user and
group id to have the resulting file permissions correct:

```sh
docker build --network host \
       --build-arg GID=$(id -g) \
       --build-arg=UID=$(id -u) \
       -t arm-rust-build .
```

Finally, to cross-compile just run:

```sh
docker run --network host -it --rm \
       -v "$(pwd)":"/home/docker/kvm-rust" \
       arm-rust-build \
       cargo build --release --target=aarch64-unknown-linux-gnu
```

## First roadblock: API difference

At first, I thought the difference would be mostly in CPU registers, so I
removed `KVM_GET_SREGS2` and `KVM_SET_SREGS2` and some x86-specific code just to
make it compile. However, when I ran it on Raspberry Pi, the program returned an
error on `KVM_GET_REGS`. I once again turned to use `strace` on QEMU to see what
was to be done. I downloaded the Linux kernel source code (from www.kernel.org),
cross-compiled it for ARM and ran it in QEMU:

```sh
#!/bin/sh

KERNEL=/home/pi/Image
qemu-system-aarch64                                             \
        -M virt                                                 \
        -smp 1                                                  \
        -enable-kvm                                             \
        -cpu host                                               \
        -kernel ${KERNEL}                                       \
        -append "console=ttyAMA0"                               \
        -serial stdio                                           \
        -nographic                                              \
        -nodefaults
```

Notice that here I used the `virt` machine specifically: that's the option to
use for KVM-based virtualization (machines like `raspi4b` would be emulated as
they require a very specific CPU and device tree, so they cannot simply run
directly on the host CPU). I put this into `start-qemu.sh` file and then ran:

```sh
strace -yy -f -X verbose -e trace=ioctl,openat,read,write,mmap -o kvm.log ./start-qemu.sh
```

### Discovering KVM register API for ARM64

First thing I did was to try and find `KVM_GET_REGS` string, but there were no
results. I tried something more generic like `grep -i regs` and got this as
first results:


```text
137152 ioctl(12<anon_inode:kvm-vcpu:0>, 0x4010aeab /* KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG */, 0x7fff57e008) = 0
137152 ioctl(12<anon_inode:kvm-vcpu:0>, 0x4010aeab /* KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG */, 0x7fff57e008) = 0
137152 ioctl(12<anon_inode:kvm-vcpu:0>, 0x4010aeab /* KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG */, 0x7fff57e008) = 0
137152 ioctl(12<anon_inode:kvm-vcpu:0>, 0x4010aeab /* KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG */, 0x7fff57e008) = 0
137152 ioctl(12<anon_inode:kvm-vcpu:0>, 0x4010aeab /* KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG */, 0x7fff57e008) = 0
137152 ioctl(12<anon_inode:kvm-vcpu:0>, 0x4010aeab /* KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG */, 0x7fff57e008) = 0
```

So this is when I realized it was going to be a bit harder. I wasn't getting
names of fields and corresponding values, just addresses. I installed Linux
headers on Raspberry Pi and searched for `KVM_[GS]ET_ONE_REG`:

```text
pi@raspberrypi:~ $ grep -Rn KVM_[GS]ET_ONE_REG /usr/include/linux/
/usr/include/linux/kvm.h:1618:#define KVM_GET_ONE_REG             _IOW(KVMIO,  0xab, struct kvm_one_reg)
/usr/include/linux/kvm.h:1619:#define KVM_SET_ONE_REG             _IOW(KVMIO,  0xac, struct kvm_one_reg)
```

Searching in the same file we can see that the `struct kvm_one_reg` is defined
as follows:

```c
struct kvm_one_reg {
       __u64 id;
       __u64 addr;
};
```

### Constructing CPU id

As it usually is with great discoveries, I learned how to construct the register
id in `kvm_one_reg` by accident. Before even knowing how KVM on ARM works with
registers, I naively tried searching for `struct kvm_regs` and found two
interesting matches:

```text
pi@raspberrypi:~ $ grep -Rn kvm_regs /usr/include/
/usr/include/aarch64-linux-gnu/asm/kvm.h:50:struct kvm_regs {
/usr/include/aarch64-linux-gnu/asm/kvm.h:208:#define KVM_REG_ARM_CORE_REG(name)(offsetof(struct kvm_regs, name) / sizeof(__u32))
```

The `struct kvm_regs` here was just a list of registers and
`KVM_REG_ARM_CORE_REG` macro seemed to be taking a register name as input and
looking for offset of the register in the `struct kvm_regs`, but I didn't find
it anywhere in the Linux headers.

I knew, however, that QEMU was one place where this might be used and I searched
for `KVM_REG_ARM_CORE_REG` and found this in [target/arm/kvm.c](https://elixir.bootlin.com/qemu/v11.0.2/source/target/arm/kvm.c#L2080):

```c
#define AARCH64_CORE_REG(x)   (KVM_REG_ARM64 | KVM_REG_SIZE_U64 | \
                 KVM_REG_ARM_CORE | KVM_REG_ARM_CORE_REG(x))
```

What was left to do was just to find these macro definitions and reconstruct
them in my Rust KVM code:

```rs
const KVM_REG_ARM64 : u64 = 0x6000000000000000;
const KVM_REG_SIZE_U64 : u64 = 0x0030000000000000;
const KVM_REG_ARM_COPROC_SHIFT : u64 = 16;
const KVM_REG_ARM_CORE : u64 = 0x0010 << KVM_REG_ARM_COPROC_SHIFT;
```

However, the `KVM_REG_ARM_CORE_REG` needed to be more thought-through as it
worked with offsets in the `struct kvm_regs`. So I have written a function
taking the register name as string and mapping it to the register `id` that KVM
understands:

```rs
fn AARCH64_CORE_REG(name: &str) -> io::Result<u64> {
    let base = KVM_REG_ARM64 | KVM_REG_SIZE_U64 | KVM_REG_ARM_CORE;
    if name.starts_with("x") {
        let reg : u64 = name[1..]
            .parse()
            .map_err(|e| io::Error::new(io::ErrorKind::InvalidInput, e))?;
        return Ok(base | reg * 2); }

    match name {
        "sp" => { return Ok(base | (31 * 2)); },
        "pc" => { return Ok(base | (32 * 2)); },
        "pstate" => { return Ok(base | (33 * 2)); },
        _ => ()
    };

    Err(io::Error::other("Invalid register."))
}
```

In my implementation I have improved this further by utilizing `HashMap` and
`LazyLock` so that the parsing and calculation isn't done on-the fly but rather
computed once and cached for subsequent calls.

## Final thoughts

At this point I understood how ARM64 exposes CPU registers through
`KVM_GET_ONE_REG` and `KVM_SET_ONE_REG` calls and had reproduced the register
`id` encoding in Rust. Unfortunately, that wasn't enough to make the hypervisor
work. My first attempt at accessing the program counter (`PC` register)
immediately failed with an `ENOEXEC` error: a clue that I was missing another
crucial step in KVM ARM setup.

The code showcasing creation of register `id`s in a `HashMap` and subsequent
error when calling `KVM_SET_ONE_REG` can be found on my GitHub page:

https://github.com/StjepanPoljak/kvm-rust/tree/kvm-arm-part1-code
