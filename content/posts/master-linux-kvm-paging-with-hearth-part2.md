+++
title = 'Mastering KVM Page Faults with Hearth: Shadow Paging, BPF and Doom'
date = 2026-11-06T17:50:53+01:00
+++

![Hearth shadow fault](/images/hearth-paging-part2.png)

## Recap

In the [previous](../master-linux-kvm-paging-with-hearth-part2) article, we
traced page faults in the host Linux kernel to understand how TDP
(Two-Dimensional Paging) works, specifically with Intel EPT (Extended Page
Tables).

EPT is a hardware mechanism that translates GPA (Guest Physical Address) into
HPA (Host Physical Address). Userspace (typically QEMU) backs guest physical
memory with host virtual memory, and KVM uses that backing when it builds the
EPT mappings. The CPU uses EPT to perform the GPA to HPA translation. We can
turn EPT off; in that case, KVM falls back to shadow paging.

## Tracing with eBPF

Before we start tracing the host, we should turn off EPT (thus making the kernel
fall back to shadow paging):

```sh
modprobe -r kvm_intel
modprobe kvm_intel ept=0
```

Also, as before, we are using the same eBPF program:

```bpftrace
#!/usr/bin/env bpftrace

tracepoint:kvm:kvm_page_fault {
        @in_kvm[tid] = args->fault_address;
}

fentry:handle_mm_fault /@in_kvm[tid]/ {
        printf("HVA=0x%lx fault_address=0x%llx\r\n", args->address, @in_kvm[tid] );
        printf("%s\r\n", kstack);
        delete(@in_kvm[tid]);
}

END { clear(@in_kvm); }
```

### Touch guest with paging on

So running Hearth with the
[Touch](https://github.com/StjepanPoljak/Hearth/blob/main/guests/touch) guest,
with paging on, gives the following results:

```text
HVA=0x750137a00000 fault_address=0x100000000

        bpf_prog_ca7b0187d5dc8911_fentry_vmlinux_handle_mm_fault_2+416
        bpf_trampoline_6442572126+79
        handle_mm_fault+5
        get_user_pages_unlocked+227
        hva_to_pfn+607
        kvm_follow_pfn+145
        __kvm_faultin_pfn+91
        __kvm_mmu_faultin_pfn+234
        kvm_mmu_faultin_pfn+346
        paging64_page_fault+279
        kvm_mmu_do_page_fault+290
        kvm_mmu_page_fault+133
        kvm_handle_page_fault+167
        vmx_handle_page_fault+97
        handle_exception_nmi+932
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

Well, we can definitely see the difference from the EPT run. Here, we got
`fault_address` as 0x100000000, while with EPT, for the same trace, we got
0x200000. So, is it really GPA as before? Well, it definitely isn't. Recall
that our Touch guest maps memory as follows:

| Guest physical          | Guest virtual                 | Size             |
|-------------------------|-------------------------------|------------------|
| `0x0`   `0x1FFFFF`      | `0x0`   `0x1FFFFF`            | 2 MiB (identity) |
| `0x200000`   `0x200FFF` | `0x100000000`   `0x100000FFF` | 4 KiB            |

The important difference is the meaning of `fault_address`. With EPT, KVM
handles an EPT violation using the GPA. With shadow paging, the hardware raises
a page fault while using the shadow page tables, and KVM obtains the faulting
virtual address from CR2. By contrast, `args->address` in `handle_mm_fault` is
a HVA: the host userspace virtual address backing the guest page.

Here's the basic idea behind shadow paging:

* The guest writes its desired page-table root to CR3.
* KVM intercepts and virtualizes that operation.
* KVM maintains a corresponding shadow page-table hierarchy.
* When entering the guest, the CPU uses the shadow page-table root rather than the guest's original CR3 value.
* KVM keeps the shadow tables synchronized with the guest's tables.

So, KVM must monitor changes to the guest's page tables and keep its shadow page
tables synchronized, requiring additional VM exits and page-table updates that
increase complexity and reduce performance.

### NMI exception

One thing is a bit puzzling: we have `handle_exception_nmi` in our trace. NMI is
a non-maskable interrupt on x86, but this function basically does more than just
handle an NMI - it also handles variety of exceptions - including page faults.

Recall how we had `handle_ept_violation` in the EPT run. The same code now
simply used a different callback:

```c
static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
        [EXIT_REASON_EXCEPTION_NMI]           = handle_exception_nmi,
        [EXIT_REASON_EXTERNAL_INTERRUPT]      = handle_external_interrupt,
        /* omitted lots of callbacks */
        [EXIT_REASON_EPT_VIOLATION]            = handle_ept_violation,
        /* ... */
```

### Touch guest with paging off

But what if we turned paging off in the guest? Recall that with EPT we didn't
really see any difference. However, with shadow paging on, results are a bit
more different:

```text
HVA=0x7f247de00000 fault_address=0x200000

        bpf_prog_ca7b0187d5dc8911_fentry_vmlinux_handle_mm_fault_2+416
        bpf_trampoline_6442572126+79
        handle_mm_fault+5
        get_user_pages_unlocked+227
        hva_to_pfn+607
        kvm_follow_pfn+145
        __kvm_faultin_pfn+91
        __kvm_mmu_faultin_pfn+234
        kvm_mmu_faultin_pfn+346
        direct_page_fault+131
        nonpaging_page_fault+18
        kvm_mmu_do_page_fault+290
        kvm_mmu_page_fault+133
        kvm_handle_page_fault+167
        vmx_handle_page_fault+97
        handle_exception_nmi+932
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

As for the stack trace, there is only slight difference from the paging on stack
trace:

|       paging on         |        paging off       |
|-------------------------|-------------------------|
| paging64_page_fault+279 | direct_page_fault+131   |
|                         | nonpaging_page_fault+18 |

### Kernel code and TDP MMU

The dispatch is performed by `kvm_mmu_do_page_fault`. In this kernel version,
its source is here:

https://elixir.bootlin.com/linux/v7.1.3/source/arch/x86/kvm/mmu/mmu_internal.h#L340

Specifically, in this part:

```c
	if (IS_ENABLED(CONFIG_MITIGATION_RETPOLINE) && fault.is_tdp)
		r = kvm_tdp_page_fault(vcpu, &fault);
	else
		r = vcpu->arch.mmu->page_fault(vcpu, &fault);
```

For the shadow-paging runs, `fault.is_tdp` is false, so KVM dispatches through
`vcpu->arch.mmu->page_fault`. The selected MMU context supplies
`paging64_page_fault` when guest paging is enabled and `nonpaging_page_fault`
when it is disabled. In the EPT runs, the TDP page-fault handler is selected;
that handler then chooses between the dedicated TDP MMU implementation and
`direct_page_fault`, depending on `tdp_mmu_enabled`. The retpoline condition in
the snippet is an optimization for the common TDP case, not the whole policy
that chooses the guest paging mode.

### TDP MMU

Now, if we have a `fault.is_tdp`, the branch will take us to the
[kvm_tdp_page_fault](https://elixir.bootlin.com/linux/v7.1.3/source/arch/x86/kvm/mmu/mmu.c#L4946) function:

```c
int kvm_tdp_page_fault(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault)
{
#ifdef CONFIG_X86_64
	if (tdp_mmu_enabled)
		return kvm_tdp_mmu_page_fault(vcpu, fault);
#endif

	return direct_page_fault(vcpu, fault);
}
```

The TDP MMU is KVM's dedicated software implementation for managing the
hardware's second-level page tables (EPT on Intel, NPT on AMD). It is possible
to keep EPT enabled while disabling this particular implementation with the
`tdp_mmu` module parameter. In that configuration, KVM still uses EPT for the
GPA-to-HPA translation, but handles faults and builds the second-level mappings
through the legacy/direct MMU path (`direct_page_fault`) instead of the TDP MMU
implementation. Note that `tdp_mmu=0` does not turn off EPT. We can try that
with:

```sh
modprobe -r kvm_intel kvm
modprobe kvm tdp_mmu=0
modprobe kvm_intel ept=1
```

Then running the same eBPF trace with Hearth will give us:

```text
HVA=0x7392e1400000 fault_address=0x200000

        bpf_prog_ca7b0187d5dc8911_fentry_vmlinux_handle_mm_fault_2+416
        bpf_trampoline_6442572126+79
        handle_mm_fault+5
        get_user_pages_unlocked+227
        hva_to_pfn+607
        kvm_follow_pfn+145
        __kvm_faultin_pfn+91
        __kvm_mmu_faultin_pfn+234
        kvm_mmu_faultin_pfn+346
        direct_page_fault+131
        kvm_tdp_page_fault+52
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

EPT is still enabled in both cases. The difference is which KVM software path
manages the second-level mappings:

|       TDP MMU on        |       TDP MMU off       |
|-------------------------|-------------------------|
| kvm_mmu_faultin_pfn+346 | kvm_mmu_faultin_pfn+346 |
| kvm_tdp_page_fault+131  |  direct_page_fault+131  |
|                         |  kvm_tdp_page_fault+52  |


## Experiments with shadow paging
Following the same approach as in the previous article, here is the summary of
trace experiments combining writing to unmapped memory address and writing to a
read-only page, all while alternating with guest paging being on and off:

| Case                        | Paging | Addresses traced (as logged)                        | Hearth printed                                   |
|-----------------------------|--------|-----------------------------------------------------|--------------------------------------------------|
| `0x200000` in RAM | on     | `0x101000`, `0x106000`, `0x100000000` (GVAs)              | nothing (reset)                                  |
| `0x200000` in RAM | off    | `0x200000` (linear address = GPA)                                      | nothing (reset)                                  |
| `0x10000000`, no RAM | on  | `0x101000`, `0x106000`, `0x100000000` (stale GVAs)         | `Unclaimed MMIO write of 4 bytes at 0x10000000.` |
| `0x10000000`, no RAM | off | `0x10000000` (stale; linear address = GPA)                             | same                                             |
| `0x200000`, read-only | on | `0x101000`, `0x106000`, `0x100000000` (stale GVAs)         | `Unclaimed MMIO write of 4 bytes at 0x200000.`   |
| `0x200000`, read-only | off | `0x200000` (stale; linear address = GPA)                               | same                                             |

Every case ended in a reset through the i8042.

## Benchmarks

To test how these various settings affect performance I decided to try benchmarking Doom without EPT (i.e. with shadow paging), with EPT and with EPT,
but with TDP MMU disabled.

### Doom results

| Mode | demo1 | demo2 | demo3 |
|---|---:|---:|---:|
| shadow paging (`ept=0`) | 2558 ± 43 | 2715 ± 34 | 2123 ± 24 |
| EPT, TDP MMU (`ept=1 tdp_mmu=1`) | 2552 ± 14 | 2703 ± 32 | 2111 ± 23 |
| EPT, old MMU (`ept=1 tdp_mmu=0`) | 2563 ± 22 | 2685 ± 26 | 2136 ± 22 |

Well, I was a bit surprised here: all three are more or less the same. I guess
Doom was bad choice for a guest that I would benchmark under shadow paging:
it builds one identity map in 2 MiB pages at boot, loads CR3 once, and runs in a
single address space with no processes. Once the tables are filled in, at start-up, KVM's MMU has nothing left to do.

### Paging bench guest

As Doom failed to produce meaningful results, I designed a benchmark that will
do something a bit more heavy with paging: forks, execs and context switches.
The source code is available on:

https://github.com/StjepanPoljak/hearth/blob/main/guests/paging-bench/bench.c

The results:

| Mode | fork + exec | fork, 64 COW faults | page fault | pipe round trip |
|---|---:|---:|---:|---:|
| shadow paging (`ept=0`) | 1724 ± 31 | 1583 ± 86 | 12.13 ± 0.21 | 31.96 ± 0.77 |
| EPT, TDP MMU (`ept=1 tdp_mmu=1`) | 394 ± 8 | 260 ± 7 | 2.46 ± 0.06 | 7.17 ± 0.12 |
| EPT, old MMU (`ept=1 tdp_mmu=0`) | 397 ± 14 | 259 ± 9 | 2.43 ± 0.02 | 7.07 ± 0.11 |

So here we have it: shadow paging is 4.5 to 6 times slower.
Forks, execs and page faults involve frequent guest page-table modifications.
Since KVM write-protects guest page-table pages to maintain shadow mappings,
these modifications can trigger VM exits and instruction emulation, adding significant overhead.

The pipe round trip, four system calls and two process switches, is hit as hard (especially due to Meltdown mitigations).

The performance difference between the two EPT implementations was negligible in
our benchmarks. The benefits of the dedicated TDP MMU implementation may become
more apparent with larger VMs and greater vCPU concurrency - something Hearth
currently lacks.

## Conclusion

With EPT disabled, KVM uses shadow page tables to combine the guest's GVA-to-GPA
translation with the host backing, producing a GVA-to-HPA mapping for the CPU.
That changes what the KVM page-fault tracepoint reports: with guest paging
enabled, the fault address is a GVA. With EPT enabled, the fault address
observed for an EPT violation is a GPA.

The `tdp_mmu=0` experiment is a useful control: EPT remains enabled, but KVM
uses the legacy/direct MMU fault path instead of the dedicated TDP MMU path.

The benchmarks told us even more. Doom was an example where shadow paging had
little measurable impact on this particular workload: it runs in a single
address space and doesn't stress context switching or page-table modifications.
The difference was running a Linux guest with a lot of context switches, forks
and execs.

The code for Hearth is available at:

https://github.com/StjepanPoljak/Hearth
