+++
title = 'Learning KVM by Reverse-Engineering QEMU with strace'
date = 2026-06-05T20:14:58+02:00
+++

![KVM trace](/images/kvm-rust-part1.png)

## Motivation

I work with virtual machines in QEMU/KVM environment (a lot). In order to debug,
optimize and customize the VMs requires an in-depth knowledge of both QEMU and
KVM, the Linux kernel virtualization subsystem that exposes hardware
virtualization features such as Intel VT-x and AMD-V to userspace applications
like QEMU. Not only that, but I work on a lot of hobby projects requiring quick
bare-metal boot-ups and debugging workflows, and, to be honest, a lot of times
QEMU is an overkill for these sorts of tasks.

## KVM vs TCG

Also when it comes to QEMU, it's worth noting that we always have an option of
using TCG, which has a completely different purpose than KVM. TCG is short for
Tiny Code Generator, which works by translating guest instructions into host
instructions at runtime. This is quite slow compared to running code without
translation overhead. So, if we want to test our bare-metal code, we may want to
test it on our own real CPU at native speed. This is where KVM comes in. Unlike
TCG, KVM does not emulate the CPU itself. Instead, it allows guest code to
execute directly on the host processor while the Linux kernel manages
transitions between guest and host execution.

## How KVM works in Linux

So, I do already know some basics. KVM driver exposes a driver interface in
Linux root filesystem, `/dev/kvm`. Communicating with the driver is done via
`ioctl()` system call on a file descriptor. What we need to find out is how QEMU
communicates with Linux kernel and try and follow the QEMU logic without
reading QEMU source code and KVM API (both can be a bit more intimidating than
just seeing how it works under the hood).

## Reverse-engineering KVM

Now we can take some lightweight Debian Linux image and load it into the QEMU,
with KVM enabled:

```sh
QEMU_IMAGE=./debian-12-nocloud-amd64.qcow2

qemu-system-x86_64										\
	-m 1024												\
	-drive file="${QEMU_IMAGE}",if=virtio,cache=none	\
	-serial stdio										\
	-enable-kvm											\
	-cpu host											\
	-nodefaults											\
	-nographic
```

This is a pretty straightforward way to run QEMU with minimal setup. The most
relevant options for us are `-enable-kvm` and `-cpu host`, which will enable
KVM and use host CPU instead of emulating some specific CPU.

### Tracing QEMU/KVM with strace

Now, we want to see what QEMU is really doing by utilizing `strace`. We can put this command in a `start-qemu.sh` script and call it with:

```sh
strace -yy -f -X verbose								\
	   -e trace=ioctl,openat,read,write,mmap			\
	   -o kvm.log										\
	   ./start-qemu.sh
```

This command will trace all `ioctl`, `openat`, `read`, `write` and `mmap` system
calls. Although I mentioned only `ioctl` calls so far, I always like to include
some other common system calls that could be used. As far as we know `/dev/kvm`
is the interface to KVM driver and QEMU will probably use `openat` on it.
Similarly, we also want to see what QEMU is doing with memory and what it's
reading and writing in general.

Note: Information on `strace` arguments as above can be found in `strace --help`
or `man strace`, but essentially, the `-yy` tells strace to print all available
information when decoding file descriptors, `-f` follows forks (we need this one
as we're wrapping it in scripts and QEMU might also do similar stuff). The
`-X verbose` will print names of constants and flags (very important when
analyzing `ioctl` calls);

### Interpreting the logs

Now, we start the above command and, as soon as system boots, we can kill it
with `CTRL+C`. This will be quite sufficient to see how QEMU/KVM works without
spamming our logs with redundant information. When we read the `kvm.log` file,
we will see a lot of traces that are not really interesting. However, we already
have some knowledge: we know QEMU should be opening `/dev/kvm` so a quick
search for `kvm` reveals exactly what we need:

```text
140900 openat(-100 /* AT_FDCWD */</home/stjepan/Develop/KVM>, "/dev/kvm", 0x80002 /* O_RDWR|O_CLOEXEC */) = 3</dev/kvm<char 10:232>>
140900 ioctl(3</dev/kvm<char 10:232>>, 0xae00 /* KVM_GET_API_VERSION */, 0) = 12
140900 ioctl(3</dev/kvm<char 10:232>>, 0xae03 /* KVM_CHECK_EXTENSION */, 0x88 /* KVM_CAP_IMMEDIATE_EXIT */) = 1
140900 ioctl(3</dev/kvm<char 10:232>>, 0xae03 /* KVM_CHECK_EXTENSION */, 0xa /* KVM_CAP_NR_MEMSLOTS */) = 32764
140900 ioctl(3</dev/kvm<char 10:232>>, 0xae03 /* KVM_CHECK_EXTENSION */, 0x76 /* KVM_CAP_MULTI_ADDRESS_SPACE */) = 2
140900 ioctl(3</dev/kvm<char 10:232>>, 0xae01 /* KVM_CREATE_VM */, 0) = 9<anon_inode:kvm-vm>
140900 ioctl(9<anon_inode:kvm-vm>, 0xae03 /* KVM_CHECK_EXTENSION */, 0x9 /* KVM_CAP_NR_VCPUS */) = 4
140900 ioctl(3</dev/kvm<char 10:232>>, 0xae03 /* KVM_CHECK_EXTENSION */, 0x42 /* KVM_CAP_MAX_VCPUS */) = 4096
```

We can see that QEMU is opening `/dev/kvm` and that it's checking API version
and various extensions. We may skip these checks and focus on the calls that
look most important; one of these here is `KVM_CREATE_VM` which also returns a
file descriptor `9<anon_inode:kvm-vm>` which we can use as a further reference.

#### Setting up memory regions

We know QEMU must eventually load firmware and guest memory into the VM. Looking
for file operations after `KVM_CREATE_VM`, we quickly encounter SeaBIOS being
loaded:

```text
140900 openat(-100 /* AT_FDCWD */</home/stjepan/Develop/KVM>, "/usr/share/seabios/bios-256k.bin", 0 /* O_RDONLY */) = 12</usr/share/seabios/bios-256k.bin>
140900 mmap(NULL, 2359296, 0 /* PROT_NONE */, 0x22 /* MAP_PRIVATE|MAP_ANONYMOUS */, -1, 0) = 0x776900dc1000
140900 mmap(0x776900e00000, 262144, 0x3 /* PROT_READ|PROT_WRITE */, 0x32 /* MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS */, -1, 0) = 0x776900e00000
140900 openat(-100 /* AT_FDCWD */</home/stjepan/Develop/KVM>, "/usr/share/seabios/bios-256k.bin", 0 /* O_RDONLY */) = 12</usr/share/seabios/bios-256k.bin>
140900 mmap(NULL, 266240, 0x3 /* PROT_READ|PROT_WRITE */, 0x22 /* MAP_PRIVATE|MAP_ANONYMOUS */, -1, 0) = 0x776900fc0000
140900 read(12</usr/share/seabios/bios-256k.bin>, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 262144) = 262144
```

We can see QEMU opening and reading SeaBIOS binary and reserving memory along
with it. We can use the return of the `read` system call (the size of the
SeaBIOS binary) and see what KVM is doing with it. Searching through the log
for this size gives us further information on what KVM is doing with SeaBIOS:

```text
140900 ioctl(9<anon_inode:kvm-vm>, 0x4020ae46 /* KVM_SET_USER_MEMORY_REGION */, {slot=3, flags=0x2 /* KVM_MEM_READONLY */, guest_phys_addr=0xfffc0000, memory_size=262144, userspace_addr=0x776900e00000}) = 0
```

So we see that it's now setting this as the memory region for KVM at guest
physical address `0xfffc0000` from userspace address that was actually
obtained by `mmap` in one of the traces above. In other words, KVM does not
allocate guest RAM itself; userspace applications such as QEMU remain
responsible for managing the backing memory.

#### Creating vCPU and running

Now, it gets very busy in the logs, but most of the stuff we see is still just
checking for extensions and capabilities. However, if we take a look at the
tail of the log, we will see a lot of these `ioctl` calls:

```text
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xae80 /* KVM_RUN */, 0) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xaeb7 /* KVM_SMI */, 0) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xae80 /* KVM_RUN */, 0) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xae80 /* KVM_RUN */, 0) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xae80 /* KVM_RUN */, 0) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xae80 /* KVM_RUN */, 0) = 0
140904 ioctl(10<anon_inode:kvm-vcpu:0>, 0xaeb7 /* KVM_SMI */, 0) = 0
```

Now both `KVM_RUN` and `KVM_SMI` are operating on a `kvm-vcpu` file descriptor,
something we haven't yet seen. So if we search the logs for it, we can actually
see where it's created:

```text
140904 ioctl(9<anon_inode:kvm-vm>, 0xae41 /* KVM_CREATE_VCPU */, 0) = 10<anon_inode:kvm-vcpu:0>
```

## Conclusion

Now we have a more complete picture of how QEMU is setting up KVM. First,
`/dev/kvm` is opened to obtain a file descriptor representing the KVM subsystem.
From it we create a new virtual machine and get `kvm-vm` file descriptor. On
this file descriptor we are setting up memory regions and later use it to create
a vCPU, on which we can call `KVM_RUN`. The following diagram explains it
better:

```text
/dev/kvm → kvm fd
    |
    +--> KVM_CREATE_VM → kvm-vm fd
            |
            +--> KVM_SET_USER_MEMORY_REGION
            |
            +--> KVM_CREATE_VCPU → kvm-vcpu fd
                            |
                            +--> KVM_RUN (loop)
```

As we can see from the logs, `KVM_RUN` appears repeatedly during guest
execution, while occasional `KVM_SMI` calls inject System Management Interrupts
into the guest. This repeated interaction between userspace and KVM is what
ultimately drives virtual CPU execution.

Next time we will recreate this exact behavior in Rust and also see about just
a few missing pieces to get our first virtual machine running in KVM.
