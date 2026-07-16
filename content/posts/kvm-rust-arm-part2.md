+++
title = 'Porting my Rust KVM hypervisor to ARM64: Running the binary'
date = 2026-07-09T19:30:54+02:00
draft = true
+++

![Successful binary run](/images/kvm-rust-arm-part2.png)

## Recap

In the [previous article](../kvm-rust-arm-part1) we ported the register handling
of my Rust KVM hypervisor from x86 to ARM64. By tracing QEMU with strace and
inspecting the Linux headers, we discovered that ARM64 accesses CPU state
through the `KVM_GET_ONE_REG` and `KVM_SET_ONE_REG` interface and reconstructed
the register `id` encoding in Rust.

At that point the hypervisor compiled successfully, but trying to access even
the program counter resulted in an `ENOEXEC` error. In this article we'll
continue the reverse-engineering process, initialize the vCPU correctly, and
finally execute our first ARM64 guest.

## Second roadblock: ENOEXEC when getting / setting PC

I was rather optimistic this would work out of the box, but even after trying
to do something as simple as reading the `PC` register I hit a snag:

```text
ioctl(5<anon_inode:kvm-vcpu:0>, KVM_ARM_SET_DEVICE_ADDR or KVM_GET_ONE_REG, 0x7fdf2c81e8) = -1 ENOEXEC (Exec format error)
Error: Os { code: 8, kind: Uncategorized, message: "Exec format error" }
```

Luckily, I had a concrete lead now: error code and KVM command. So I searched
for the `KVM_GET_ONE_REG` in the Linux kernel code on Bootlin Elixir and found
the following code in [arch/arm64/kvm/arm.c](https://elixir.bootlin.com/linux/v6.1.21/source/arch/arm64/kvm/arm.c#L1298):

```c
	case KVM_SET_ONE_REG:
	case KVM_GET_ONE_REG: {
		struct kvm_one_reg reg;

		r = -ENOEXEC;
		if (unlikely(!kvm_vcpu_initialized(vcpu)))
			break;
```

This is exactly the error we were getting and we immediately found what was
missing: vCPU initialization. Actually, if we take a closer look at the code we
will see the comment kernel developers have left us:

```c
int kvm_arch_vcpu_create(struct kvm_vcpu *vcpu)
{
	int err;

	/* Force users to call KVM_ARM_VCPU_INIT */
	vcpu->arch.target = -1;
```

### Initializing the vCPU

Following the lead I looked for `KVM_ARM_VCPU_INIT` in the original `strace` log
and found:

```text
137152 ioctl(12<anon_inode:kvm-vcpu:0>, 0x4020aeae /* KVM_ARM_VCPU_INIT */, 0x7fff57e028) = 0
137156 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4020aeae /* KVM_ARM_VCPU_INIT */, 0x7f82f70e68) = 0
137152 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4020aeae /* KVM_ARM_VCPU_INIT */, 0x7fff57dd78) = 0
137152 ioctl(10<anon_inode:kvm-vcpu:0>, 0x4020aeae /* KVM_ARM_VCPU_INIT */, 0x7fff57e048) = 0
```

This wasn't really helpful: we again only see addresses, but we see no hints of
underlying structure. So let's first take a look at how `KVM_ARM_VCPU_INIT` is
defined in the Linux headers:

```text
$ grep -Rn KVM_ARM_VCPU_INIT /usr/include/
/usr/include/linux/kvm.h:1622:#define KVM_ARM_VCPU_INIT   _IOW(KVMIO,  0xae, struct kvm_vcpu_init)
```

A quick search for `struct kvm_vcpu_init` revealed:

```text
$ grep -Rn 'struct kvm_vcpu_init' /usr/include/ -A3
/usr/include/aarch64-linux-gnu/asm/kvm.h:112:struct kvm_vcpu_init {
/usr/include/aarch64-linux-gnu/asm/kvm.h-113-   __u32 target;
/usr/include/aarch64-linux-gnu/asm/kvm.h-114-   __u32 features[7];
/usr/include/aarch64-linux-gnu/asm/kvm.h-115-};
```

### Getting struct data with Python GDB

I didn't want to spend much time understanding every target and feature flag. My
goal was simply to reproduce QEMU's behavior as quickly as possible (and I
didn't even have to understand it completely). Luckily, this structure was
really simple (basically an array of eight `32`-bit integers), so I decided to
tackle this with Python GDB automation. I created a breakpoint to stop exactly
on `ioctl` call when we are in `KVM_ARM_VCPU_INIT`:

```py
import gdb
import struct

def log(line):
    print(line)
    with open("gdb-debug.out", "a") as f:
        f.write(line + "\n")

class IoctlBreakpoint(gdb.Breakpoint):

    def stop(self):
        if x1 == 0x4020aeae: # KVM_ARM_VCPU_INIT
            ptr = int(gdb.parse_and_eval("$x2"))
            vcpu_init_mem = gdb.selected_inferior().read_memory(ptr, 32) # u32 + 7 * u32
            target_features = struct.unpack("<8I", bytes(vcpu_init_mem))
            log(f"KVM_ARM_VCPU_INIT: target={target_features[0]} features={target_features[1:]}")        
            return False

if __name__ == "__main__":
    tracer = IoctlBreakpoint("ioctl")
    gdb.execute("handle SIGUSR1 SIGUSR2 nostop noprint pass")
    gdb.execute("run")
```

I ran my QEMU command with GDB:

```sh
gdb -ex 'source ./trace-arm.py' ./start-qemu.sh
```

This has automatically yielded great results:

```text
KVM_ARM_VCPU_INIT: target=5 features=(0, 0, 0, 0, 0, 0, 0)
KVM_ARM_VCPU_INIT: target=5 features=(12, 0, 0, 0, 0, 0, 0)
KVM_ARM_VCPU_INIT: target=5 features=(12, 0, 0, 0, 0, 0, 0)
KVM_ARM_VCPU_INIT: target=5 features=(12, 0, 0, 0, 0, 0, 0)
```

For now I want to try just setting `target`, but leave `features` zeroed out. I
could immediately reconstruct this in Rust:

```rs
impl VCPU {
    pub fn arm_vcpu_init(&self) -> io::Result<()> {
        let mut vcpu_init : kvm_vcpu_init = unsafe { std::mem::zeroed() };
        // KVM_ARM_VCPU_INIT: target=5 features=(0, 0, 0, 0, 0, 0, 0)
        vcpu_init.target = 5;
        let ret = unsafe { libc::ioctl(self.fd, KVM_ARM_VCPU_INIT, &mut vcpu_init) };
        if ret < 0 {
            return Err(io::Error::last_os_error());
        }
        Ok(())
    }
}
```

## Third roadblock: No KVM exit

Soon after this I was able to get and set registers properly. The first ARM test
binary I compiled was based on working UART code from my hobby operating system
I started on Raspberry Pi. I expected at least a fault exit, if not an MMIO exit
when UART code was called. However, `KVM_RUN` never returned.

This is where I was stuck for a while. I considered creating a separate thread
that repeatedly queried `KVM_GET_ONE_REG` for `PC`, but from my own working
experience I knew that calling KVM `ioctl` calls asynchronously was a really bad
idea. I also considered adding hardware breakpoint capabilities to at least
trigger `KVM_EXIT_DEBUG`, but this would require more infrastructure for just
checking simple binary.

### Writing a simple binary

I decided to try adding a signal handler instead. If I interrupted `KVM_RUN`
call with `SIGINT` maybe I could get registers then synchronously. I lowered
my expectations for initial binary and just created something really simple:

```asm
.section .text

.macro curr_el_to reg
	mrs	\reg, CurrentEL
	lsr	\reg, \reg, #2
	and	\reg, \reg, #0xf
.endm

.globl _start
_start:
	mrs	x0, mpidr_el1
	and	x0, x0, #0xFF
	cbz	x0, control
	b .

control:
	curr_el_to x0
	b .
```

It looks like much, but this code will just load current CPU privilege level to
`x0` register (being that level `EL=1` is operating-system level, `x0` should be
set to `0x1`). The `mpidr_el1` code is for checking if we are on control CPU
(this should be unnecessary here, but I added this out of paranoia caused by
real-life issues I had on ARM).

### Installing signal handler

Finally, I added a signal handler:

```rs
extern "C" fn handler(_sig: libc::c_int) { }

unsafe fn install_interrupt_signal() {
    let mut sa: libc::sigaction = std::mem::zeroed();
    sa.sa_sigaction = handler as *const() as usize;
    libc::sigemptyset(&mut sa.sa_mask);
    sa.sa_flags = 0;
    libc::sigaction(libc::SIGINT, &sa, std::ptr::null_mut());
}
```

The handler itself does nothing. I only needed the signal to interrupt the blocking `KVM_RUN` call. So my final KVM loop looked like:

```rs
    unsafe { install_interrupt_signal() };

    let run = vcpu.kvm_run_mem as *mut kvm_run;

    loop {
        let ret = unsafe { libc::ioctl(vcpu.fd, KVM_RUN, 0usize) };
        if ret < 0 {
            if io::Error::last_os_error().raw_os_error() == Some(libc::EINTR) {
                vcpu.print_regs()?;
            }
            return Err(io::Error::last_os_error());
        }

        let exit_reason = unsafe { (*run).exit_reason };
```

## Running the binary

Finally, it was time to run the gues:

```sh
pi@raspberrypi:~ $ ./rust hello-world-arm.img
x0 = 0x0
pc = 0x1000
Loading "hello-world-arm.img" to 0x1000...
^Cx0 = 0x1
pc = 0x101c
Error: Os { code: 4, kind: Interrupted, message: "Interrupted system call" }
```

We can see that `PC` did change and that `x0` really is set to current CPU
privilege level: `0x1`. Why it didn't work for my UART code is probably that the
guest itself requires more initialization than I though (x86 went more smoothly
and felt more plug-and-play).

## Conclusion

Although x86 and ARM64 share the same KVM interface at a high level, their
userspace APIs differ significantly. Porting my hypervisor turned out to involve
much more than renaming registers: ARM64 uses per-register `id`s instead of a
single register structure, requires explicit vCPU initialization through
`KVM_ARM_VCPU_INIT` and demanded a fair amount of reverse engineering before I
could successfully execute guest code. Fortunately, the same workflow that
worked on x86 like combining strace, GDB and the Linux kernel source proved just
as effective on ARM64 (although required a bit more effort).
