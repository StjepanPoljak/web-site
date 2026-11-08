+++
title = 'Mastering KVM Page Faults with Hearth: Discovering EPT with bpftrace'
date = 2026-10-30T20:02:58+01:00
+++

![Hearth page fault](/images/hearth-paging-part1.png)

## Motivation

Paging on Linux can be difficult to understand for newcomers. There's a lot of
stuff going on. Add virtualization on top and things get even more confusing:
how do guest page tables and KVM interact to translate guest virtual addresses
into guest physical addresses, and how does KVM map those guest physical
addresses to host memory? And what actually happens when one of these
translations fails?

The goal of this article is to understand how Linux and the CPU hardware itself
handle memory translation in virtual machine environments.

## The Touch guest

I wanted to create a minimal guest whose sole purpose would be to access a
specific memory address. I started with a small x86 assembly program that writes
a value to address `0x200000`.

The program is compiled as an ELF file with a Xen PVH note, allowing us to reuse
our existing Linux [PVH boot](../notes-on-x86-pvh-boot) code. Recall that PVH
boot starts us in 32-bit protected mode. From there, we can set up our own
page tables and have a controlled environment to experiment with memory
translation.

Note that we have identity-mapped first 2 MiB with a single page mapped to
0x100000000 (4 GiB offset):

| Guest physical          | Guest virtual                 | Size             |
|-------------------------|-------------------------------|------------------|
| `0x0` – `0x1FFFFF`      | `0x0` – `0x1FFFFF`            | 2 MiB (identity) |
| `0x200000` – `0x200FFF` | `0x100000000` – `0x100000FFF` | 4 KiB            |

We did this so we can see the difference in what the trace will report.

Complete code for Touch is available in my GitHub repository:

https://github.com/StjepanPoljak/Hearth/blob/main/guests/touch

## Tracing the guest

My first attempt to get some trace / starting point was running Touch in Hearth
while tracing any function containing `page_fault` in its name. My initial
assumption was that the guest will definitely trigger a page fault on the host,
as this memory is managed by Linux kernel as ordinary userspace memory.

```sh
sudo bpftrace \
	 -c './target/debug/hearth --linux build/touch/touch.elf' \
	 -e 'fentry:*page_fault* { print(kstack); }'
```

This gives us one possible stack trace to start with:

```text
        bpf_prog_80d58c72ad362cd6_fentry_kvm_is_page_fault_stale_11+135
        bpf_trampoline_2123861499894+71
        is_page_fault_stale+5
        kvm_mmu_do_page_fault+558
        kvm_mmu_page_fault+133
        handle_ept_violation+249
        __vmx_handle_exit+573
        vmx_handle_exit+18
        vt_handle_exit+26
        vcpu_enter_guest+2190
        vcpu_run+70
        kvm_arch_vcpu_ioctl_run+849
        kvm_vcpu_ioctl+777
        __x64_sys_ioctl+163
        x64_sys_call+4155
        do_syscall_64+261
        entry_SYSCALL_64_after_hwframe+118
```

Not bad for first attempt, and most of these functions we can understand from
their name alone. However, one is a bit weird: `handle_ept_violation`. Wait, we
violated what? More on that in the next chapter.

#### Host userspace page faults

Now just to see the difference, this is the "normal" host userspace page fault
trace:

```sh
$ sudo bpftrace \
    -c './target/debug/hearth --linux build/touch/touch.elf' \
    -e 'fentry:vmlinux:handle_mm_fault {
            printf("handle_mm_fault: 0x%lx\r\n", args->address); print(kstack); }'
handle_mm_fault: 0xb3409e1e000

        bpf_prog_f32ba2c4cb858a2f_fentry_vmlinux_handle_mm_fault_1+243
        bpf_trampoline_6442572126+79
        handle_mm_fault+5
        exc_page_fault+131
        asm_exc_page_fault+39
```

Here we see a regular host userspace page fault. The CPU raises a page-fault
exception, which enters the Linux exception-handling path and eventually reaches
`handle_mm_fault()`.

### Second-level address translation

Before we continue we need to clarify what EPT here is. EPT stands for
"Extended Page Tables" and it's basically solving in hardware what older shadow
paging solved in software (but more on that later).

With EPT there are two independent translation stages. The guest owns the
first one: it sets CR3 and builds its own page tables to translate a guest
virtual address (GVA) to a guest physical address (GPA). KVM owns the second
one: it points the CPU at the EPT (via the EPT pointer in the VMCS) to translate
GPA to host physical address (HPA). The CPU walks both in hardware, and KVM
gets involved when an EPT violation occurs.

AMD has its own equivalent called NPT (Nested Page Tables). KVM uses the term
TDP (Two-Dimensional Paging) when referring to this kind of second-level
translation, regardless of whether the underlying hardware is Intel or AMD.

#### Disabling EPT

On Intel, EPT can be disabled with:

```sh
modprobe -r kvm_intel
modprobe kvm_intel ept=0
```

On AMD, just substitute `kvm_amd` and `npt` in favor of `kvm_intel` and `ept`,
respectively.

Disabling EPT means KVM can no longer rely on hardware-assisted second-level
translation and instead has to use shadow paging. This is something we'll
investigate in more detail later.

#### Handling EPT violations

An EPT violation occurs when a guest memory access cannot be completed because
the EPT mapping is missing or its permissions don't allow the requested
access. The CPU generates a VM exit, which KVM handles.

Depending on the situation, KVM may install or update an EPT mapping, or handle
the access through another path, such as MMIO emulation.

Once a valid EPT mapping is available, subsequent accesses can use it directly
without triggering another EPT violation.

## Following a KVM memory fault

Now, getting back at our trace, I was browsing through these functions to see if
some of them are in `tracepoint` in bpftrace - that would give us ready to use
arguments without going deep into the kernel code. Very soon I found this one:

```text
$ sudo bpftrace -lv 'tracepoint:kvm:kvm_page_fault'
tracepoint:kvm:kvm_page_fault
    unsigned int vcpu_id
    unsigned long guest_rip
    u64 fault_address
    u64 error_code
```

And yes, the `fault_address` sounds like something I'm looking for. Now to run
this under Hearth gives us rather useful information:

```sh
$ sudo bpftrace \
    -c './target/debug/hearth --linux build/touch/touch.elf' \
    -e 'tracepoint:kvm:kvm_page_fault {
            printf("kvm_page_fault: 0x%llx\r\n", args->fault_address); }'
Attached 1 probe

Hearth
RAX=0x0 RBX=0x10000     RCX=0x0 RDX=0x600
RSI=0x0 RDI=0x0 RSP=0x0 RBP=0x0
R8=0x0  R9=0x0  R10=0x0 R11=0x0
R12=0x0 R13=0x0 R14=0x0 R15=0x0
RIP=0x100000    RFLAGS=0x2
kvm_page_fault: 0x100000
kvm_page_fault: 0x101000
Guest reset through the i8042.
kvm_page_fault: 0x102000
kvm_page_fault: 0x103000
kvm_page_fault: 0x104000
kvm_page_fault: 0x105000
kvm_page_fault: 0x106000
kvm_page_fault: 0x200000
```

In the log above we see the address we are writing to: `0x200000`. However,
contrary to my assumption, we also got more page faults from our guest. The
range from `0x101000` to `0x106000` is easy: that's basically our guest zeroing
out the `.bss` section where our page tables live.

But what about `0x100000`? Well, when we load the program into the guest memory,
we are actually writing to the host memory - so we get a "classical" page fault
which we cannot see here. But when the guest sets the RIP to `0x100000` and
starts fetching instructions, it also touches the memory and causes an EPT
violation. This is why we see `0x100000` in our trace as well.

## Connecting GPA and HVA

For now we traced only KVM-specific page faults. But let's try and trace the
more generic page faults with `handle_mm_fault` and connect them with our
`kvm_page_fault`:

```text
$ sudo bpftrace -lv 'fentry:vmlinux:handle_mm_fault'
fentry:vmlinux:handle_mm_fault
    struct vm_area_struct * vma
    long unsigned int address
    unsigned int flags
    struct pt_regs * regs
    vm_fault_t retval
```

Here we have `address`, but that's the host virtual address (HVA), not guest
physical address (GPA) we have seen so far. To connect these, I have written a
more elaborate BPF program for bpftrace:

```bpftrace
#!/usr/bin/env bpftrace

tracepoint:kvm:kvm_page_fault {
        @in_kvm[tid] = args->fault_address;
}

fentry:handle_mm_fault /@in_kvm[tid]/ {
        printf("HVA=0x%lx GPA=0x%llx\r\n",
               args->address, @in_kvm[tid] );
        printf("%s\r\n", kstack);
        delete(@in_kvm[tid]);
}

END { clear(@in_kvm); }
```

Whenever we go into `kvm_page_fault` it will remember GPA via `fault_address`
tracepoint argument and store it by the key `tid` in `in_kvm` map. Then when
(and if) we get to `handle_mm_fault`, we will print both GPA and HVA, along
with a kernel stack to see what's going on.

A note about the `@in_kvm` map: I later noticed that my bpftrace program
doesn't remove entries from the `@in_kvm` map when a corresponding
`handle_mm_fault` isn't reached. This means some entries can remain around and
potentially be matched with a later host page fault. I discovered this after
running more experiments, but didn't bother changing the script, since I
wasn't particularly interested in refining this type of trace. The important
part here is the KVM fault addresses and the behaviour observed by Hearth.

An example result of this trace on Hearth:

```text
HVA=0x74dc83a00000 GPA=0x200000

        bpf_prog_ca7b0187d5dc8911_fentry_vmlinux_handle_mm_fault_2+416
        bpf_trampoline_6442572126+79
        handle_mm_fault+5
        get_user_pages_unlocked+227
        hva_to_pfn+607
        kvm_follow_pfn+145
        __kvm_faultin_pfn+91
        __kvm_mmu_faultin_pfn+234
        kvm_mmu_faultin_pfn+346
        kvm_tdp_page_fault+131
        kvm_mmu_do_page_fault+558
        kvm_mmu_page_fault+133
        handle_ept_violation+249
        __vmx_handle_exit+573
        vmx_handle_exit+18
        vt_handle_exit+26
        vcpu_enter_guest+2190
        vcpu_run+70
        kvm_arch_vcpu_ioctl_run+849
        kvm_vcpu_ioctl+777
        __x64_sys_ioctl+163
        x64_sys_call+4155
        do_syscall_64+261
        entry_SYSCALL_64_after_hwframe+118
```

### Diving into the kernel code

Going through the kernel code, we can see the logic here. We are in `vcpu_run`
function (this is a function we simply end up when we call `KVM_RUN` ioctl) and
ended up in `vt_handle_exit`. It wasn't straightforward to see the connection
here, but `vt_handle_exit` is actually called via a callback in
[vcpu_enter_guest](https://elixir.bootlin.com/linux/v7.1.3/source/arch/x86/kvm/x86.c#L11613):

```c
static int vcpu_enter_guest(struct kvm_vcpu *vcpu)
{
  /* lots of code omitted */
        r = kvm_x86_call(handle_exit)(vcpu, exit_fastpath);
        return r;
```

Then from `vmx_handle_exit` we get to the
[kvm_vmx_exit_handlers](https://elixir.bootlin.com/linux/v7.1.3/source/arch/x86/kvm/vmx/vmx.c#L6339)
table where our `handle_ept_violation` is listed:

```c
/*
 * The exit handlers return 1 if the exit was handled fully and guest execution
 * may resume. Otherwise they set the kvm_run parameter to indicate what needs
 * to be done to userspace and return 0.
 */
static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
        [EXIT_REASON_EXCEPTION_NMI]           = handle_exception_nmi,
        [EXIT_REASON_EXTERNAL_INTERRUPT]      = handle_external_interrupt,
        /* omitted lots of callbacks */
        [EXIT_REASON_EPT_VIOLATION]            = handle_ept_violation,
        /* ... */
```

Note that we didn't really see a KVM exit in userspace: the comment of this
callback table explains it perfectly - some KVM exits are handled by the host
kernel, some may inform userspace. So, this table takes us directly to
`handle_ept_violation`. Note that this is Intel VMX-specific code. Eventually,
this code will lead to more generic `kvm_tdp_page_fault` where we will follow
`hva_to_pfn` (Host Virtual Address to Page Frame Number) path until we again
end up in `handle_mm_fault`.

## Experimenting with EPT mappings

Closing in on the article, I did some experiments combining writing to unmapped
memory address and writing to a read-only page, all while alternating with guest
paging being on and off.

| Case                        | Paging | GPAs traced                                | Hearth printed                                   |
|-----------------------------|--------|--------------------------------------------|--------------------------------------------------|
| `0x200000`, in RAM | on     | `0x101000` - `0x106000`, `0x200000`          | nothing (reset)                                  |
| `0x200000`, in RAM | off    | `0x200000`                                 | nothing (reset)                                  |
| `0x10000000`, outside of RAM     | on     | `0x101000` - `0x106000`, `0x10000000` | `Unclaimed MMIO write of 4 bytes at 0x10000000.` |
| `0x10000000`, outside of RAM     | off    | `0x10000000`                        | same                                             |
| `0x200000`, read-only    | on     | `0x101000` - `0x106000`, `0x200000`   | `Unclaimed MMIO write of 4 bytes at 0x200000.`   |
| `0x200000`, read-only    | off    | `0x200000`                          | same                                             |

Two kinds of stack appear:

| Kind                         | Stack, abbreviated                                                                                                 | HVA                                        |
|------------------------------|--------------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| guest fault, filled by KVM   | `handle_ept_violation`   `kvm_tdp_page_fault`   `kvm_faultin_pfn`   `hva_to_pfn`   `get_user_pages_unlocked`   `handle_mm_fault` | in guest RAM, page aligned, base + GPA      |
| host fault in Hearth | `asm_exc_page_fault`   `exc_page_fault`   `handle_mm_fault` | outside guest RAM, ends in ` 008`          |

In our experiments, we don't see a significant difference between having guest
paging enabled or disabled. However, keep in mind that we're tracing KVM's
handling of memory faults, not the guest's own page-table activity. The
`kvm_page_fault` tracepoint tells us about faults handled by KVM, but doesn't
expose the guest's GVA-to-GPA translations.

Also note that with paging off we don't see `0x101000` - `0x106000` simply
because zeroing out `.bss` was not compiled in the binary.

## Conclusion

We started with a minimal guest that simply writes to a memory address and
followed that access through KVM and into the Linux memory-management subsystem.

Using bpftrace, we were able to observe EPT violations, trace the guest physical
address and connect it to the host virtual address backing guest memory. We
also saw how KVM handles these faults internally, without necessarily returning
to userspace.

This is only one side of KVM memory translation. EPT provides hardware-assisted
second-level translation, while shadow paging takes a different approach by
having KVM maintain shadow page tables. That's something we'll explore next.
