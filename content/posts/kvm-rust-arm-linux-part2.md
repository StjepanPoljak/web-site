+++
title = 'Booting ARM64 Linux in my Rust KVM Hypervisor: Getting to the console with help of eBPF'
date = 2026-09-11T18:42:32+02:00
draft = true
+++

## Recap
In the [previous article](../kvm-rust-arm-linux-part1) I got ARM64 Linux as far
as executing inside my Rust KVM hypervisor. I still couldn't actually interact
with the guest, though: I had no working console.

This article is about getting from "the PC is moving" to typing "hello" into
Linux and getting "HELLO" back.

## UART

Last time we learned how to obtain the device tree blob from QEMU's `virt`
machine. Now we want to take a look at what's inside. We can decompile it:

```sh
dtc -I dtb -O dts -o virt.dts virt.dtb
```

Then, a simple search through the file already gives us the clue we need:

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

So, now I had a lead: I needed to emulate the `pl011` UART at `0x09000000` so
that the existing Linux `pl011` driver could communicate with it. This was not a
really interesting part, as most of it was simply reading Linux kernel source
code along with `pl011` documentation from ARM. Soon I was able to get output
from Linux and it felt really great:

```text
[    0.000000] Linux version 7.1.3 (docker@stjepan-Aspire-F5-573G) (aarch64-linux-gnu-gcc (Debian 10.2.1-6) 10.2.1 20210110, GNU ld (GNU Binutils for Debian) 2.35.2) #1 SMP PREEMPT Thu Jul 16 08:52:24 UTC 2026
[    0.000000] KASLR disabled on command line
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] efi: UEFI not found.
[    0.000000] OF: reserved mem: Reserved memory: No reserved-memory node in the DT
[    0.000000] NUMA: Faking a node at [mem 0x0000000040000000-0x0000000047ffffff]
[    0.000000] NODE_DATA(0) allocated [mem 0x47fc10c0-0x47fc37bf]
                  ### omitted further kernel log ###
```

## Creating custom initramfs

Now I needed bidirectional UART support so that I could send input to the guest
as well. To make further investigation easier, I created a very simple binary
written in `C` to be run as the Linux init process: it just prints out current
interrupt statistics and asks user for input, echoing it back in all caps. All
we then need is `initramfs.txt` file with some simple folders and devices:

```text
dir / 0755 0 0
dir /dev 0755 0 0
dir /proc 0555 0 0
dir /sys 0555 0 0
nod /dev/kmsg 0600 0 0 c 1 11
nod /dev/console 0600 0 0 c 5 1
file /init /home/stjepanp/Develop/initramfs/init 0755 0 0
```

I only needed enough of a filesystem to provide `/init`, `/dev/console`, and
access to `/proc` for interrupt statistics. Then, in the Linux source code
folder, we generate `initramfs` with:

```sh
./usr/gen_initramfs.sh ../initramfs/initramfs.txt -o initramfs
```

To see at which memory address to add `initramfs`, we can run ordinary QEMU with
our `initramfs` and see what happens:

```sh
KERNEL=/home/pi/Image
qemu-system-aarch64                                             \
        -M virt,dumpdtb=virt.dtb                                \
        -enable-kvm                                             \
        -cpu host                                               \
        -kernel ${KERNEL}                                       \
        -append "console=ttyAMA0 nokaslr rdinit=/init"          \
        -serial stdio                                           \
        -nographic                                              \
        -nodefaults                                             \
        -initrd "./initramfs"
```

After decompiling `virt.dtb` (explained in [previous](../kvm-rust-arm-linux-part1) article) we can see the location:

```text
pi@raspberrypi:~/dtb $ grep initrd virt-initramfs.dts -B2 -A3
        chosen {
                bootargs = "console=ttyAMA0 initramfs_async=0 rdinit=/init nokaslr";
                linux,initrd-start = <0x0 0x40200000>;
                linux,initrd-end = <0x0 0x40294200>;
                stdout-path = "/pl011@9000000";
                kaslr-seed = <0x8114b902 0xe815aa93>;
        };
```

We do have to recalculate `initrd-end` if size of our `initramfs` file changes.
If that happens, we can easily recompile the device tree blob after changing
the value. Also, let's not forget to load `initramfs` in Rust code:

```rs
const LOAD_ADDR: u64 = 0x40_000_000;
const INITRAMFS_OFFS: u64 = 0x200_000;
const KERN_OFFS: u64 = 0x200_000 + INITRAMFS_OFFS;

impl VM {
    pub fn arch_load_linux(&mut self, vcpu: &mut VCPU, args: &Args) -> io::Result<()> {
        let linux_mem_idx = self.add_mem_region(1024 * 1024 * 1024, LOAD_ADDR)?;
        self.load_file_to_memory(linux_mem_idx, &dtb_path, 0x0)?;
        self.load_file_to_memory(linux_mem_idx, &initramfs_path, INITRAMFS_OFFS)?;
		// ...
    }
```

## Setting up vGIC

Even without having the ability to send keystrokes there was one big problem
with the Linux console `stdout`: The kernel messages were reaching the console,
but once execution reached `/init`, output stopped behaving as expected. It
took me some time to realize that the ARM architectural timer was also involved.
Its interrupts need to be delivered through the virtual GIC, so I first needed
to create a vGIC. It turned out creating the vGIC was easy. I just followed the
logic from QEMU source code:

https://elixir.bootlin.com/qemu/v11.0.1/source/accel/kvm/kvm-all.c#L4035

In Rust this translates to:

```rs
let mut create_device: kvm_create_device = unsafe { std::mem::zeroed() };
create_device.type_ = KVM_ARM_DEVICE_VGIC_V2;
create_device.fd = 0xffffffff;
create_device.flags = 0;

unsafe { libc::ioctl(self.fd, KVM_CREATE_DEVICE, &mut create_device); }
```

This worked. Also note that I have omitted proper error handling to keep the
code short and concise.

### Tracing IRQ injection

Now if we run our `initramfs` with `strace` as before, and we try to type in
some keys, we will see that in our log we have:

```text
pi@raspberrypi:~ $ grep -i irq kvm.log | tail -n5
458398 ioctl(9<anon_inode:kvm-vm>, 0x4008ae61 /* KVM_IRQ_LINE */, 0x7fd932dbd0) = 0
458403 ioctl(9<anon_inode:kvm-vm>, 0x4008ae61 /* KVM_IRQ_LINE */, 0x7f85c1db70) = 0
458403 ioctl(9<anon_inode:kvm-vm>, 0x4008ae61 /* KVM_IRQ_LINE */, 0x7f85c1db30) = 0
458403 ioctl(9<anon_inode:kvm-vm>, 0x4008ae61 /* KVM_IRQ_LINE */, 0x7f85c1db20) = 0
458403 ioctl(9<anon_inode:kvm-vm>, 0x4008ae61 /* KVM_IRQ_LINE */, 0x7f85c1db70) = 0
```

So our `ioctl` for IRQ injection actually takes an address as argument. If we
investigate this, we can find out how `KVM_IRQ_LINE` is handled in the Linux
kernel:

https://elixir.bootlin.com/linux/v6.1.21/source/virt/kvm/kvm_main.c#L4776

Reading this code we see that the relevant function is `kvm_vm_ioctl_irq_line`
that takes `struct kvm_irq_level` as second argument and this is what we need
to interpret:

```c
/* for KVM_IRQ_LINE */
struct kvm_irq_level {
	/*
	 * ACPI gsi notion of irq.
	 * For IA-64 (APIC model) IOAPIC0: irq 0-23; IOAPIC1: irq 24-47..
	 * For X86 (standard AT mode) PIC0/1: irq 0-15. IOAPIC0: 0-23..
	 * For ARM: See Documentation/virt/kvm/api.rst
	 */
	union {
		__u32 irq;
		__s32 status;
	};
	__u32 level;
};
```

For this purpose, we can treat it as an 8-byte structure containing the IRQ number and level. Therefore, we can easily inspect it again via eBPF when we
run QEMU with our `initramfs`:

```text
pi@raspberrypi:~ $ sudo bpftrace -e 'kprobe:kvm_vm_ioctl_irq_line { printf("irq_level=%x\n", *((uint64 *)arg1)); }'
Attaching 1 probe...
irq_level=1000021
irq_level=1000021
irq_level=1000021
irq_level=1000021
```

Without wasting much time we can easily reconstruct this behavior in Rust:

```rs
pub fn arch_update_irq(level: u32, vm_fd: libc::c_int) -> io::Result<()> {
    let mut irq: kvm_irq_level = unsafe { std::mem::zeroed() };
    irq.level = level;
    irq.__bindgen_anon_1.irq = 0x1000021;

    let ret = unsafe {
        libc::ioctl(vm_fd, KVM_IRQ_LINE, &mut irq)
    };

    if ret < 0 {
        return Err(io::Error::last_os_error());
    }

    Ok(())
}
```

Note that, in essence, `0x1000021` is not an arbitrary magic number. On ARM64,
`KVM_IRQ_LINE` encodes the interrupt type and interrupt ID in the `irq` field. Here `0x01` selects an SPI and `0x21` is interrupt 33—the UART interrupt from
the device tree.

### No such device or address

However, when we run our program, it fails with "No such device or address". To
get some clue on what is going on, we can try `strace` and get:

```text
ioctl(5, KVM_RUN, 0)                    = -1 ENXIO (No such device or address)
Error: Os { code: 6, kind: Uncategorized, message: "No such device or address" }
```

My first impulse was to search for `ENXIO` in Elixir bootlin, but,
unfortunately, there were simply too many functions and branches that could
return `ENXIO`. Rather than trying to follow every possible `ENXIO` path
manually, I decided to use a more brute-force approach. I attached `kretprobe`s
to the KVM functions and looked for one returning `-ENXIO`:

```text
pi@raspberrypi:~ $ sudo bpftrace -e 'kretprobe:kvm_* /retval == (uint32)(-6)/ { printf("%s\n", probe); }'
Attaching 487 probes...
kretprobe:kvm_vgic_map_resources
kretprobe:kvm_arch_vcpu_run_pid_change
```

The second function here does not seem really relevant, so I tried looking for
`kvm_vgic_map_resources` on Elixir:

https://elixir.bootlin.com/linux/v6.1.21/source/arch/arm64/kvm/vgic/vgic-v2.c#L289

Now taking a look at the function we can clearly see where `ENXIO` is returned:

```c
int vgic_v2_map_resources(struct kvm *kvm)
{
    struct vgic_dist *dist = &kvm->arch.vgic;
    int ret = 0;

    if (IS_VGIC_ADDR_UNDEF(dist->vgic_dist_base) ||
        IS_VGIC_ADDR_UNDEF(dist->vgic_cpu_base)) {
        kvm_debug("Need to set vgic cpu and dist addresses first\n");
        return -ENXIO;
    }
    // ...
```

Now we have a hint. We need to set vGIC CPU and distributor addresses first. To
confirm our hypothesis we can try tracing this function specifically:

```text
pi@raspberrypi:~ $ sudo bpftrace -e 'kretprobe:vgic_v2_map_resources { printf("ret=%u\n", retval); }'
Attaching 1 probe...
ret=4294967290
```

### Finding out distributor address

My first attempt to set up distributor address didn't go well. I wrote the
following code (I extracted distributor address from the device tree and
tried to reconstruct the logic from QEMU source code):

```rs
let dist_addr: u64 = 0x08_000_000;
let mut dev_attr: kvm_device_attr = unsafe { std::mem::zeroed() };
dev_attr.group = KVM_DEV_ARM_VGIC_GRP_ADDR;
dev_attr.attr = KVM_VGIC_V2_ADDR_TYPE_DIST;
dev_attr.flags = 0;
dev_attr.addr = dist_addr;

unsafe { libc::ioctl(create_device.fd as i32, KVM_SET_DEVICE_ATTR, &mut dev_attr) };
```

However, when I ran the program, I got:

```text
ioctl(6, KVM_SET_DEVICE_ATTR, 0x7ffe2696d8) = -1 EFAULT (Bad address)
```

To debug this, I needed to inspect the values defined in
`struct kvm_device_attr`, but it was a lot more complicated: I had to check if
the `struct` I was tracing was for `KVM_DEV_ARM_VGIC_GRP_ADDR` group and for
`KVM_VGIC_V2_ADDR_TYPE_DIST`. So, I decided to write my eBPF program in C with
and called it `probe-kvm.bpf.c`:

```c
#include <linux/bpf.h>
#include <linux/types.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <linux/kvm.h>
#include <stdbool.h>
#include <stdint.h>

char LICENSE[] SEC("license") = "GPL";

SEC("kprobe/vgic_v2_set_attr")
int BPF_KPROBE(kprobe_vgic_v2_set_attr, void *dev, struct kvm_device_attr *dev_attr)
{
        struct kvm_device_attr attr;
        __u64 value;

        bpf_probe_read_kernel(&attr, sizeof(attr), dev_attr);
        if (attr.group != KVM_DEV_ARM_VGIC_GRP_ADDR) {
                return 0;
        }

        if ((attr.attr != KVM_VGIC_V2_ADDR_TYPE_DIST) && (attr.attr != KVM_VGIC_V2_ADDR_TYPE_CPU)) {
                return 0;
        }

        bpf_printk("\nset_attr: attr=%s addr=0x%llx", attr.attr == KVM_VGIC_V2_ADDR_TYPE_DIST ? "DIST" : "CPU", attr.addr);

        return 0;
}
```

To compile and run it I wrote the following script:

```sh
#!/bin/sh

clang -O2 -g -mcpu=v1 -target bpf -I /usr/include/aarch64-linux-gnu -c probe-kvm.bpf.c -D__TARGET_ARCH_arm64 -o probe-kvm.bpf.o
sudo rm -rf /sys/fs/bpf/vgic
sudo bpftool prog loadall probe-kvm.bpf.o /sys/fs/bpf/vgic autoattach
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

So, I started ordinary QEMU along my eBPF trace and got this output:

```text
$ ./start-probe.sh
           <...>-146994  [002] d..31 953951.139387: bpf_trace_printk: 
set_attr: attr=CPU addr=0x7fd4d3c450

           <...>-146994  [002] d..31 953951.139418: bpf_trace_printk: 
set_attr: attr=DIST addr=0x7fd4d3c450
```

This really looked like a userspace address while I was supplying a raw address.
To find out what was at that address I modified the last line of the BPF program
above:

```c
        if (bpf_probe_read_user(&value, sizeof(value), (void *)attr.addr) != 0)
                return -1;

        bpf_printk("\nset_attr: attr=%s addr=0x%llx, *addr=0x%llx\n", attr.attr == KVM_VGIC_V2_ADDR_TYPE_DIST ? "DIST" : "CPU", attr.addr, value);

```

With this I got what I didn't really expect:

```text
           <...>-147143  [000] d..31 954397.183145: bpf_trace_printk: 
set_attr: attr=CPU addr=0x7fdc806ca0, *addr=0x8010000

           <...>-147143  [000] d..31 954397.183177: bpf_trace_printk: 
set_attr: attr=DIST addr=0x7fdc806ca0, *addr=0x8000000
```

So the `addr` field is actually a pointer to the address, and not the address
itself. Note that `0x8010000` and `0x8000000` are the guest physical
addresses of the GIC CPU interface and distributor taken from the virt machine's
device tree:

```sh
pi@raspberrypi:~/dtb $ grep -i intc virt.dts -A3
        intc@8000000 {
                phandle = <0x8001>;
                reg = <0x00 0x8000000 0x00 0x10000 0x00 0x8010000 0x00 0x10000>;
                compatible = "arm,cortex-a15-gic";
```

This also explained the `EFAULT`: KVM was trying to dereference
`attr.addr` as a userspace pointer. With this small fix, I got a fully working
Linux console:

```rs
dev_attr.addr = &dist_addr as *const u64 as u64;
```

The following snippet shows that my `init` process (as part of
`initramfs`) was run and it echoed back my input in capital letters:

```text
[    0.832623] clk: Disabling unused clocks
[    0.834046] PM: genpd: Disabling unused power domains
[    0.845565] Freeing unused kernel memory: 12672K
[    0.847475] Run /init as init process
           CPU0       
 11:        199 GIC-0  27 Level     arch_timer
 13:          0 GIC-0  33 Level     uart-pl011
IPI0:         0       Rescheduling interrupts
IPI1:         0       Function call interrupts
IPI2:         0       CPU stop interrupts
IPI3:         0       CPU stop NMIs
IPI4:         0       Timer broadcast interrupts
IPI5:       168       IRQ work interrupts
IPI6:         0       CPU backtrace interrupts
IPI7:         0       KGDB roundup interrupts
Err:          0

type a string: hello
allcaps: HELLO

[    2.224776] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000000
[    2.227382] CPU: 0 UID: 0 PID: 1 Comm: init Not tainted 7.1.3 #1 PREEMPT 
[    2.229749] Hardware name: linux,dummy-virt (DT)
```

The interrupt statistics are also printed from my `init`. Also, when `init`
exits, we get the (correct) Kernel panic that init process was terminated.

## Final thoughts

At this point I finally had what I was looking for: a real Linux userspace
running inside my own KVM-based hypervisor, with a working UART, interrupt
delivery and a minimal `initramfs`.

More importantly, I now had a way to interact with the guest and inspect what
was happening inside KVM. That gives me a much better foundation for the next
step: implementing more of the virtual hardware needed to turn this into a
useful ARM64 virtual machine.

You can also check out the repository on my GitHub page and, if you have a
Raspberry Pi 4B, try the VMM yourself:

https://github.com/StjepanPoljak/kvm-rust
