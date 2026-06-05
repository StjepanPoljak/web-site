+++
title = 'Automating Stack Corruption Analysis in GDB with Python'
date = 2026-05-23T19:52:31+02:00
draft = false
+++

![Python GDB](/images/python-gdb.png)

## A bug in my operating system

During a recent visit to my wife's family in Sarajevo, I decided to revisit my
hobby operating system in QEMU. I discovered that the boot process consistently
froze while printing the BIOS memory map. What initially looked like a
protected-mode issue eventually turned into a useful exercise in automating
debugging using GDB's Python scripting.

## Manual debugging failure

After reinspecting some common pitfalls in my protected mode setup I was more
confident that the issue was stack corruption in my `print_memory_map` function.
If I had an unmatched push or pop, it could have corrupted return addresses and
eventually redirected execution flow into invalid memory. Single-stepping
through the routine manually quickly became impractical. The function mixed BIOS
interrupt handling, memory map parsing, and multiple helper calls, making it
difficult to reason about stack state over time.

## Setting up GDB scripting in Python

I needed to automate this and GDB's integration with Python was the most
promising route I could take. The idea was to do exactly what I started
manually: break at a specific (suspicious) function and then start single
stepping while inspecting how the stack pointer behaved.

First things first, we import GDB module in Python, connect to the remote
target and set up a breakpoint (this is done by inheriting from
`gdb.Breakpoint`):

```py
import gdb

class StackTraceBreakpoint(gdb.Breakpoint):

    def __init__(self, func_name):
        super(StackTraceBreakpoint, self).__init__(func_name)
        self.active = False
    
    def stop(self):
        if self.active:
            raise Exception("Recursion detected - stopping.")
        self.active = True

        return True

if __name__ == "__main__":
    gdb.execute("target remote :5555")
    gdb.execute("symbol-file ../build/arch/x86/bios-legacy/boot-stage1-5.elf")

    tracer = StackTraceBreakpoint("print_memory_map")
    gdb.execute("continue")
```

### StackTraceBreakpoint

So this is kind of bare-bones of what I wanted to do. This code will simply add
a breakpoint with custom logic after connecting to QEMU and loading symbols.
As soon as we do `gdb.execute("continue")` GDB will run and, if and when it hits
our breakpoint, it will execute whatever we wrote in the `stop()` method. For
now I only added a kind of assertion that we cannot analyze recursions (I didn't
use any in my code anyway and logic would be a bit more complex).

### Single-stepping

Now what we need to do is start single-stepping after we hit our breakpoint. So
we add a while loop with `gdb.execute("stepi")` after `gdb.execute("continue")`.
Note that the `stepi` instruction steps over machine instructions, not over
source code statements. Also note that We cannot start single-stepping in
`stop()` method because GDB won't be in a state which can accept these kinds of
debugging requests.

## Detecting stack imbalance

Furthermore I have wrapped this single-stepping logic in `trace_step()` method
in our breakpoint class. This method is not part of GDB breakpoint API, but
rather as a convenience for tracking the number of pushes and pops in a
consistent manner. To run this script we need to call:

```sh
gdb -ex 'source debug-stack.py'
```

Another thing we need to track is whether we have entered another function (in
which case we won't be counting pushes and pops) and if we have returned from
it. If I ever wanted to inspect routines being called, I would just run the same
script for them (my intention here is not creating a custom emulator on top of
GDB). So I added a counter `func_count` which will increase on `call` and
decrease on `ret` instruction. Here is a rough idea:

```py
def trace_step(self):
	insn_full = gdb.execute("x/i $pc", to_string=True).strip()
	insn = re.search(r'^=>.*:\s*([^\s].*)$', insn_full).group(1).split(" ")[0]

	match insn:
		case "push":
			if self.func_count == 0:
				self.pushes += 1
		case "pop":
			if self.func_count == 0:
				self.pops += 1
		case "call":
			self.func_count += 1
		case "ret":
			self.func_count -= 1
```

Here you can see how I'm extracting the instruction from GDB. And what is left
is just improving logic and also fetching registers like PC, SP and CS for
debugging. I log everything into a file as GDB can get really noisy with
standard output (I didn't find a way to turn off all logging in GDB completely
when single stepping). The script is available in my GitHub repository:

https://github.com/StjepanPoljak/raspios/tree/master/scripts/debug-stack.py

## Finding the root cause

Finally, you can see an example output detecting my very issue:

```text
[START] print_memory_map SP=fffd
[0000:8183] push %ax (SP=0xfffd)
[0000:8185] push %bx (SP=0xfff9)
[0000:8187] push %cx (SP=0xfff5)
[0000:8189] push %dx (SP=0xfff1)
[0000:81a5] call 0x66e980a5 (SP=0xffed)
[0000:80a3] push %ax (SP=0xffeb)
[0000:80a7] call 0xab18056 (SP=0xffe7)
(...)
[0000:806e] ret  (SP=0xffef)
[0000:8098] call 0xf6ec8056 (SP=0xfff1)
[0000:8054] push %ax (SP=0xffef)
[0000:8056] push %bx (SP=0xffeb)
[0000:806a] pop %bx (SP=0xffe7)
[0000:806c] pop %ax (SP=0xffeb)
[0000:806e] ret  (SP=0xffef)
[0000:8098] call 0xf6ec8056 (SP=0xfff1)
[0000:8054] push %ax (SP=0xffef)
[0000:8056] push %bx (SP=0xffeb)
[0000:806a] pop %bx (SP=0xffe7)
[0000:806c] pop %ax (SP=0xffeb)
[0000:806e] ret  (SP=0xffef)
[0000:809d] pop %esi (SP=0xfff1)
[0000:809e] pop %bx (SP=0xfff3)
[0000:80a0] pop %ax (SP=0xfff7)
[0000:80a2] ret  (SP=0xfffb)
[FAIL] Extra pop detected at [0000:81fa].
```

So the real culprit was an extra `pop eax` in my `print_memory_map` routine:

```asm
print_memory_map:
	push eax
	push ebx
	push ecx
	push edx

; --- ommited print loop logic ---

.noprint_newline:
	pop eax	
	
	cmp ecx, [memory_map_size]
	jne .print_memory_map_loop

	pop edx
	pop ecx
	pop ebx
	pop eax
	ret
```

Removing this line will cause my debugging script to successfully pass.

## Try it out yourself

You can try it out yourself, just check out my operating system, `raspios`, on
GitHub:

https://github.com/StjepanPoljak/raspios

Build and run it with:

```sh
mkdir build
cd build
ARCH=x86 cmake ..
make
make qemu_debug
```

Then, in the `scripts` folder run `gdb -ex 'source debug-stack.py'`.

## Conclusion

This was a good reminder that low-level debugging often benefits from
lightweight tooling tailored to the problem at hand. In this case, a small
amount of Python automation around GDB made stack corruption analysis
significantly more manageable than manual instruction tracing.
