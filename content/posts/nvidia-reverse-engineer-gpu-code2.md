+++
title = 'Reverse-Engineering NVIDIA: Changing a register operand'
date = 2026-09-22T20:02:55+02:00
draft = false
+++

![Modifying CUDA binary](/images/nvidia-reverse2.png)

## Motivation

In the [previous article](../nvidia-reverse-engineer-gpu-code) I have
started reverse-engineering a simple CUDA binary by modifying the value of
a pointer variable.

After identifying where the value was stored and successfully running the modified CUDA binary, it was time to find out what the corresponding assembly instructions actually looked like at the machine-code level.

## Finding the immediate size

Now, to be able to determine the size and format of the instruction (or at
least the largest constant the instruction from the previous article can hold),
I started to increase the value word-by-word. Instead of simple `0xcc` from the
last time, I did:

```c
#include <stdio.h>

__global__ void test(int *a) {
    a[0] = 0xecc;
}

int main(int argc, const char* argv[]) {
    cudaError_t err;
    int *a;
    cudaMallocManaged(&a, sizeof(int));
    *a = 0x01;
    test<<<1, 1>>>(a);
    if ((err = cudaGetLastError()) != cudaSuccess)
        fprintf(stderr, "launch: %s\n", cudaGetErrorString(err));

    cudaDeviceSynchronize();
    printf("*a = %x\n", *a);

    return 0;
}
```

And yes, it was there:

```text
$ xxd -e out/test-old.sm_50.cubin | grep ecc
00000690: ecc7f000 01000000 05070002 4c980780   ...............L
```

Now to try a bit larger value like '0xdecc':

```text
$ xxd -e out/test-old.sm_50.cubin | grep decc
```

Searching for the complete `0xdecc` pattern didn't find anything. Searching for
the original `0xecc` did, however, and the surrounding bytes had changed:

```text
$ xxd -e out/test-old.sm_50.cubin | grep ecc
00000690: ecc7f000 0100000d 05070002 4c980780   ...............L
```

And there it was. If we compare this line with the hex dump from above, we can
see that we have an extra `d` in `0100000d`. So I tried increasing the word
size in `xxd`:

```text
$ xxd -e -g 8 out/test-old.sm_50.cubin | grep decc
00000690: 0100000decc7f000 4c98078005070002   ...............L
```

Now the bytes looked much more like a structured `64`-bit value rather than an
arbitrary sequence of bytes. The next thing to do was simply increase the test
value and see how much the apparent immediate field could hold.
The result with the largest value was:

```text
$ xxd -e -g 8 out/test-old.sm_50.cubin | grep ffffffff
00000690: 010ffffffff7f000 4c98078005070002   ...............L
```

After that, it was impossible to fit more into an `int` value as `nvcc`
complained about truncation. I naturally tried to find out what will
happen if I use `uint64_t` instead of `int`:

```text
$ xxd -e -g 8 out/test-old.sm_50.cubin | grep ffffffff -C1
00000680: 001fc400fe4007f1 010eeeeeeee7f004   ..@.............
00000690: 010ffffffff7f005 eedd200000070204   ............. ..
000006a0: 001f9c00fde007ef 50b0000000070f00   ...............P
```

So we discovered something. We now had two apparent `MOV`s and I wasn't sure
whether they differed because they targeted different registers or because they
loaded the high and low parts of a value.

## Adding a static variable

After seeing how the `MOV` behaved while changing the value, I tried a little
experiment. I added one more variable with a lot of bytes:

```c
#include <stdint.h>

__device__ uint64_t x = 0xdaec;

__global__ void test(uint64_t *a) {
        x = 0xccccccccdddddddd;
        a[0] = 0xaaaaaaaabbbbbbbb;
}
```

After looking for my usual patterns, I got:

```text
$ xxd -e -g 8 out/test-old.sm_50.cubin | grep bbbb -B5 -A7
00000780: 001fc400fe2007f6 4c98078000870001   .. ............L
00000790: 010000000007f002 010000000007f003   ................
000007a0: 001fc000fe4007f1 010dddddddd7f008   ..@.............
000007b0: 010cccccccc7f009 4c98078005070004   ...............L
000007c0: 001fc400fc2007f1 eedd200000070208   .. .......... ..
000007d0: 4c98078005170005 010bbbbbbbb7f006   .......L........
000007e0: 001fbc00fe2007f2 010aaaaaaaa7f007   .. .............
000007f0: eedd200000070406 50b0000000070f00   ..... .........P
00000800: 001ffc00fc4007ef 50b0000000070f00   ..@............P
00000810: 50b0000000070f00 e30000000007000f   .......P........
00000820: 001f8000fc0007ff e2400fffff07000f   ..............@.
00000830: 50b0000000070f00 50b0000000070f00   .......P.......P
00000840: 000000000000daec 0000000000000000   ................
```

Looking at this we can filter some of these out. We can filter by patterns I
created. Also, I noticed that the line beginning with `0xeed` appeared a lot
in my dumps. Here it appeared two times, so I filtered useful stuff out:

```text
$ xxd -c 8 -e -g 8 out/test-old.sm_50.cubin | grep 'bbbb\|cccc\|aaaa\|dddd\|eedd'
000007a8: 010dddddddd7f008   ........
000007b0: 010cccccccc7f009   ........
000007c8: eedd200000070208   ..... ..
000007d8: 010bbbbbbbb7f006   ........
000007e8: 010aaaaaaaa7f007   ........
000007f0: eedd200000070406   ..... ..
```

The interesting part was that the final byte changed systematically as I introduced different values. This made it look increasingly likely that the final byte represented a register operand.



This is current test output:

```text
$ ./test-old 
*a = aaaaaaaabbbbbbbb
```

Can we make it display the value of variable `x`? Let's try hacking `0xeed2`
instructions. My hypothesis was that the low bits of these instructions encode
the register operand, similarly to what we observed in the `0x10` lines:

```text
$ xxd -c 8 -e -g 8 test-old | grep eedd2
0008a848: eedd200000070208   ..... ..
0008a870: eedd200000070406   ..... ..
```

I used `bvi` to change the final byte of the second instruction from `06` to
`08`:

```text
xxd -c 8 -e -g 8 test-old | grep eedd2
0008a848: eedd200000070208   ..... ..
0008a870: eedd200000070408   ..... ..
```

And (not) surprisingly, I got:

```text
$ ./test-old
*a = ccccccccdddddddd
```

We actually switched registers from which the value for `a[0]` was loaded.

## Conclusion

For now we can identify (and modify to a certain extent) two instructions:

The first instruction appears to be a `MOV`-like instruction, with a format that
currently looks like:

```text
010VVVVVVVV7f00R
```

The `VVVVVVVV` appears to represent an immediate value, while `R` appears to
identify the destination register.

The second instruction appears to select a register containing the value that is
ultimately written to `a[0]`. Its encoding currently appears to be:

```text
eedX200000070Y0R
```

where `R` appears to identify the register. `X/Y` remain unknown; I suspect one
of them encodes information about the operand size, since I observed different
values when experimenting with 32-bit `int` versus 64-bit `uint64_t` values.
