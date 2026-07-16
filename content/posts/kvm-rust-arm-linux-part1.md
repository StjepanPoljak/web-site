+++
title = 'Booting ARM64 Linux in my Rust KVM Hypervisor: Debugging a Memory Issue'
date = 2026-07-13T16:57:18+02:00
draft = true
+++

![Function not implemented error](/images/kvm-rust-arm-linux-part1.png)

## Motivation

The time has come: the big test. I got some simple binaries running on my Rust
KVM hypervisor (at this point I am thinking about giving it a proper name). The
next thing has to be Linux. For this series I will show how to actually boot
Linux on my Rust KVM hypervisor on ARM64 CPU (I am using Raspberry Pi 4 which
supports hardware accelerated virtualization).

## Cross-compiling Linux kernel

In one of my [previous articles](../kvm-rust-arm-part1) I have described how to
set up the cross-compilation environment with a simple `Dockerfile`. We can
reuse it to cross-compile the Linux kernel for ARM64 (after adding some Linux
build dependencies like `bc`, `flex`, `python3`, `libssl-dev` and `bison`):

```sh
docker run --network host -it --rm \
       -e CROSS_COMPILE=aarch64-linux-gnu- -e ARCH=arm64 \
       -v "$(pwd)":"/home/docker/kvm-rust" -w "/home/docker/kvm-rust/linux-7.1.3" \
       arm-rust-build /bin/bash -c 'make defconfig && make'
```

The resulting ARM64 Linux kernel image (`Image`) should be located in
`arch/arm64/boot/` directory.

## ARM64 boot protocol

The full ARM64 Linux boot protocol is quite straightforward and can be found in
the Linux kernel documentation:

https://www.kernel.org/doc/html/v7.1/arch/arm64/booting.html

We simply need to parse the 64-byte header and verify the magic field (it should
equal `0x644d5241`). I will just highlight a few important points. We should
read `image_size` and `text_offset` fields and note that the image should be
loaded at an offset of text_offset bytes  at a 2 MiB boundary. For our kernel,
as I checked, `text_offset` is zero and `image_size` is the amount of memory the
decompressed kernel image occupies in RAM, which is not necessarily the same as
the file size as reported by `stat`. Finally, the registers `x1`, `x2` and `x3`
must be set to zero, while `x0` must point to the device tree blob (`dtb`) in
the RAM. So, for this I chose to load the `dtb` at address `0x40000000` and
`Image` itself on the first 2 MiB boundary:

```rs
const LOAD_ADDR: u64 = 0x40_000_000;
const KERN_OFFS: u64 = 0x200_000;
```

### Obtaining the device tree blob

So, for ARM64 we do need a device tree blob that will tell the Linux kernel on
what kind of hardware it is running on. And this step is quite easy, we will
can simply reuse our command for running QEMU, with one small addition (i.e.
adding `dumpdtb` option to `-M virt` argument):

```sh
#!/bin/sh

KERNEL=/home/pi/Image
qemu-system-aarch64                                             \
        -M virt,dumpdtb=virt.dtb                                \
        -smp 1                                                  \
        -enable-kvm                                             \
        -cpu host                                               \
        -kernel ${KERNEL}                                       \
        -append "console=ttyAMA0"                               \
        -serial stdio                                           \
        -nographic                                              \
        -nodefaults
```

This gives us a consistent snapshot of the hardware we were running on (in this 
case it's emulated hardware for QEMU `virt` machine). We could (and we should)
edit and prune it for our own purposes, but let's leave that for another time.

### Implementing boot in Rust

So, loading the kernel in Rust is quite straightforward:

```rs
impl VM {
    pub fn arch_load_linux(&mut self, vcpu: &mut VCPU, args: &Args) -> io::Result<()> {
        let header = get_linux_header(&args.binary)?;
        let linux_mem_idx = self.add_mem_region((KERN_OFFS + header.image_size) as usize, LOAD_ADDR)?;

        match &args.dtb {
            Some(dtb_path) => { self.load_file_to_memory(linux_mem_idx, &dtb_path, 0x0)?; },
            None => { return Err(io::Error::other("Device tree blob not specified.")); }
        }

        self.load_file_to_memory(linux_mem_idx, &args.binary, KERN_OFFS)?;

        vcpu.set_one_reg("x0", LOAD_ADDR)?;
        vcpu.set_one_reg("x1", 0x0)?;
        vcpu.set_one_reg("x2", 0x0)?;
        vcpu.set_one_reg("x3", 0x0)?;

        vcpu.set_one_reg("pc", LOAD_ADDR + KERN_OFFS)?;
    }
}
```

Note that the `get_linux_header()` function was omitted here as it only parses
the aforementioned ARM64 Linux image headers in a boilerplate manner.

## Function not implemented

As soon as I tried to run my hypervisor with Linux `Image`, I ran into an issue:

```text
pi@raspberrypi:~ $ ./rust -d virt Image 
x0 = 0x0
pc = 0x1000
magic 0x644d5241, image_size: 38731776, file_size: 38040064, text_offset: 0x0
Loading "virt" to 0x0...
Loading "Image" to 0x200000...
x0 = 0x40000000
pc = 0x40200000
x0 = 0xfffffbfffda35000
pc = 0xffffd902f0f8a42c
Error: Os { code: 38, kind: Unsupported, message: "Function not implemented" }
```

### Trying strace

First thing I turned to was `strace`; it's easy to use and gets some nice
information really fast:

```text
ioctl(5, KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG, 0x7fcee98768) = 0
x0 = 0x40000000
ioctl(5, KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG, 0x7fcee98768) = 0
pc = 0x40200000
ioctl(5, KVM_RUN, 0)                    = -1 ENOSYS (Function not implemented)
ioctl(5, KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG, 0x7fcee988a8) = 0
x0 = 0xfffffbfffda35000
ioctl(5, KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG, 0x7fcee988a8) = 0
pc = 0xffffd902f0f8a42c
Error: Os { code: 38, kind: Unsupported, message: "Function not implemented" }
+++ exited with 1 +++
```

I didn't learn much other than the fact that `KVM_RUN` failed completely with
`ENOSYS` error code. My first attempt to find out what was going on was
searching the ARM64 KVM source code for `ENOSYS` in Linux repository. It didn't
take long, as it appeared only in one place which was a promising lead:

https://elixir.bootlin.com/linux/v7.1.3/source/arch/arm64/kvm/mmio.c#L188

As we can see, the function in question is `io_mem_abort()` in the Linux kernel.
Now, `strace` got us that far, but this is a Linux kernel function and not much
we can find out more about it with `strace` (which traces only system calls like
`ioctl`). I haven't even considered `GDB` because this was the host kernel,
and this would require a really complex time-consuming setup. Luckily, there was
a much easier way.

### Analyzing IO memory abort with eBPF

The first thing I did was to list `kprobe`s in `bpftrace` and see if I can hook
into the `io_mem_abort`˙at all via eBPF:

```text
pi@raspberrypi:~ $ sudo bpftrace -l 'kprobe:io_mem_abort'
kprobe:io_mem_abort
```

I was in luck, so I could at least confirm that this function was the one
causing the issue, so I attached a `kretprobe` to it, ran `bpftrace` alongside
my hypervisor and got the confirmation I needed:

```text
pi@raspberrypi:~ $ sudo bpftrace -e '
>    kretprobe:io_mem_abort {
>        printf("ret=%d\n", retval);
>    }'
Attaching 1 probe...
ret=-38
```

### Finding out what really happened

Now that I have confirmed `io_mem_abort` was the culprit, it was time to
analyze it more deeply. I attached an additional `kprobe` so I could print both
the return value and the `fault_ipa` argument:

```c
int io_mem_abort(struct kvm_vcpu *vcpu, phys_addr_t fault_ipa)
```

The updated `bpftrace` script printed the fault address.

```text
pi@raspberrypi:~ $ sudo bpftrace -e '
>    kretprobe:io_mem_abort {
>        printf("ret=%d\n", retval);
>    }
>    kprobe:io_mem_abort {
>        printf("0x%llx\n", arg1);
>    }'
Attaching 2 probes...
0x47fff000
ret=-38
```

So the result surprised me as I initially suspected it was trying to access some
device that I haven't created; however, I realized that this was actually just
beyond the memory I had allocated for the Linux kernel to use. So it clicked: I probably hadn't allocated enough guest memory. Instead of allocating just
`header.image_size + KERN_OFFS` I increased it to 1 GiB:

```rs
let linux_mem_idx = self.add_mem_region(1024 * 1024 * 1024, LOAD_ADDR)?;
```

## Conclusion

With that adjustment, the `ENOSYS` error disappeared. The guest was clearly
executing code, although I still wasn't seeing any output on the serial console.
That meant the memory issue was solved, but there was at least one more thing
left before we would actually see Linux boot.





### UART

 For now we want to decompile it and see what's inside:

```sh
dtc -I dtb -O dts -o virt.dts virt.dtb
```
