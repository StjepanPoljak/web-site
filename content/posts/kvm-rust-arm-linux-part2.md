+++
title = 'Booting ARM64 Linux in my Rust KVM Hypervisor: Getting to console'
date = 2026-07-22T18:44:51+02:00
draft = true
+++

## Recap

In the [previous article](../kvm-rust-arm-linux-part1) I explained how I got the
Linux to boot on ARM64. Well, probably: with my interrupt signal handler I could
see that the program counter was moving around in high-memory, but I still had
no definite proof that the Linux really booted. I needed the console.

## UART

Last time we learned how to obtain the device tree blob from QEMU's `virt`
machine. Now we want to take a look at what's inside. We can decompile it:

```sh
dtc -I dtb -O dts -o virt.dts virt.dtb
```

Then, a simple search through the file already gives us a hint we need:

```text
pi@raspberrypi:~ $ grep -i uart virt.dts -A5 -B1
        pl011@9000000 {
                clock-names = "uartclk\0apb_pclk";
                clocks = <0x8000 0x8000>;
                interrupts = <0x00 0x01 0x04>;
                reg = <0x00 0x9000000 0x00 0x1000>;
                compatible = "arm,pl011\0arm,primecell";
				};
```

So, now I had a lead: I needed to implement the `pl011` driver which is located
at `0x9000000` base address. This was not a really interesting part, as most of
it was simply reading Linux kernel source code along with `pl011` documentation
from ARM. Soon I was able to get output from Linux and it felt really great!

## Creating custom initramfs




```text
pi@raspberrypi:~ $ sudo bpftrace -e 'kretprobe:kvm_* /retval == (uint32)(-6)/ { printf("%s\n", probe); }'
Attaching 487 probes...
kretprobe:kvm_vgic_map_resources
kretprobe:kvm_arch_vcpu_run_pid_change
```

```text
pi@raspberrypi:~ $ sudo bpftrace -e 'kretprobe:vgic_v2_map_resources { printf("ret=%u\n", retval); }'
Attaching 1 probe...
ret=4294967290
```

```text
137152 mmap(NULL, 2101248, 0 /* PROT_NONE */, 0x22 /* MAP_PRIVATE|MAP_ANONYMOUS */, -1, 0) = 0x7f81fff000
137152 mmap(0x7f82000000, 4096, 0x3 /* PROT_READ|PROT_WRITE */, 0x32 /* MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS */, -1, 0) = 0x7f82000000
137152 mmap(NULL, 2101248, 0 /* PROT_NONE */, 0x22 /* MAP_PRIVATE|MAP_ANONYMOUS */, -1, 0) = 0x7f81dff000
137152 mmap(0x7f81e00000, 4096, 0x3 /* PROT_READ|PROT_WRITE */, 0x32 /* MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS */, -1, 0) = 0x7f81e00000
137152 ioctl(11<anon_inode:kvm-arm-vgic-v2>, 0x4018aee1 /* KVM_SET_DEVICE_ATTR */, 0x55b0d0ffa0) = 0
137152 ioctl(11<anon_inode:kvm-arm-vgic-v2>, 0x4018aee1 /* KVM_SET_DEVICE_ATTR */, 0x55b0d0fb20) = 0
```
