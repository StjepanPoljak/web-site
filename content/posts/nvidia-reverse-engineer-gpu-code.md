+++
title = 'Reverse-Engineering NVIDIA: Modifying a CUDA binary'
date = 2026-09-05T18:42:28+02:00
draft = false
+++

![Modifying CUDA binary](/images/nvidia-reverse1.png)

## Motivation

For quite a while I wanted to find out how NVIDIA really works under the hood.
One of the greatest unknowns for me was how GPU code is compiled, packaged and
loaded. So I accepted my own challenge: let's try and decode machine code
running on GPU.

## Test code

I needed to start really simple and to get as much data as possible. My first
CUDA program to try and decipher was simply:

```c
#include <stdio.h>

__global__ void test(int *a) {
        a[0] = 0x00;
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

I won't get into too many details here (I am not a CUDA expert), but essentially, the function marked with `__global__` is the kernel that will run on the device. The `cudaMallocManaged()` function takes care of allocating memory that can be accessed by both the host and the device, saving us some additional memory-management code. The `test<<<1, 1>>>(a)` syntax launches the `test` kernel with one block containing one thread, while the CUDA runtime takes care of loading the kernel for execution on the GPU. Finally, `cudaDeviceSynchronize()` waits for the GPU to finish executing the kernel, ensuring that the result written to `a` is available when we access it from the host.

### Build

I also created a reusable build command that will also give me something to
start with:

```sh
nvcc -arch=sm_50 -lineinfo --keep --keep-dir out -o test test.cu
```

Note: for this to work, the `out` directory needs to be created before running
`nvcc` compiler. Now I got a folder full of really useful files:

```text
stjepan@stjepan-Aspire-F5-573G:~/Develop/NVIDIA$ ls out/
test.cpp1.ii    test.cudafe1.cpp     test_dlink.fatbin    test_dlink.reg.c        test.fatbin.c   test.ptx
test.cpp4.ii    test.cudafe1.gpu     test_dlink.fatbin.c  test_dlink.sm_50.cubin  test.module_id  test.sm_50.cubin
test.cudafe1.c  test.cudafe1.stub.c  test_dlink.o         test.fatbin             test.o
```

The `test.sm_50.cubin` file sounded like exactly what I was looking for:

```text
stjepan@stjepan-Aspire-F5-573G:~/Develop/NVIDIA$ file out/test.sm_50.cubin 
out/test.sm_50.cubin: ELF 64-bit LSB executable, NVIDIA CUDA architecture,, statically linked, not stripped
```

## First hack

Now I wanted to try changing the value of `a` in the `test` function and see how
`cubin` file will behave. I had no idea what I was dealing with, so I started
checking the file with random small values to get the one with least repetitions
in the CUDA binary. After some random attempts I realized that the best one is:

```text
stjepan@stjepan-Aspire-F5-573G:~/Develop/NVIDIA$ hexdump out/test.sm_50.cubin | grep bc
0000680 07f1 fde0 bc00 001f 02ff 0007 2000 eedc
00008f0 0000 0000 0000 0000 04bc 0000 0000 0000
```

So I decided to change my test function to:

```c
__global__ void test(int *a) {
        a[0] = 0xbc;
}
```

If everything went well, I would see an additional line here containing my `bc`.
The reason I started with really small values is because I had no idea what kind
of byte ordering I will encounter. After rebuilding the binary, I now got:

```text
stjepan@stjepan-Aspire-F5-573G:~/Develop/NVIDIA$ hexdump out/test.sm_50.cubin | grep bc
0000310 f1bc 06d4 02fc 0000 0209 0000 0000 0000
0000690 f000 0bc7 0000 0100 0002 0507 0780 4c98
00006a0 07f2 fe20 bc00 001f 0003 0517 0780 4c98
```

Well, I definitely got a bit more different hex dump. So I tried just changing
it again, this time to `0xcc` (just randomly) to see what I will get. And I got:

```text
stjepan@stjepan-Aspire-F5-573G:~/Develop/NVIDIA$ hexdump out/test.sm_50.cubin | grep cc
0000690 f000 0cc7 0000 0100 0002 0507 0780 4c98
```

This was promising. Notice that this is the same as address `0x0000690` in the
previous run, except we got `0cc7` instead of `0bc7`. Now I could try and find
this kind of pattern in the real binary:

```text
stjepan@stjepan-Aspire-F5-573G:~/Develop/NVIDIA$ hexdump test | grep 0cc7
000cc70 528b 4830 e789 5c89 1024 c748 2444 0008
008a700 0001 0087 0780 4c98 f000 0cc7 0000 0100
```

We can see our value on address `0x008a700`. Right now, if we run the binary,
we get:

```text
stjepan@stjepan-Aspire-F5-573G:~/Develop/NVIDIA$ ./test 
*a = cc
```

Now, I used `bvi` to quickly find the address and changed `0cc7` to `0dd7`.
So the result was:

```text
stjepan@stjepan-Aspire-F5-573G:~/Develop/NVIDIA$ hexdump test | grep 0dd7
000dd70 01bb 0000 e900 ff76 ffff 0f66 441f 0000
008a700 0001 0087 0780 4c98 f000 0dd7 0000 0100
```

Well, hopefully, I didn't corrupt the binary. When I tried running it:

```text
stjepan@stjepan-Aspire-F5-573G:~/Develop/NVIDIA$ ./test 
*a = dd
```

Bingo. We didn't recompile the CUDA source, regenerate the CUBIN, or even touch
the host executable's source code. We changed bytes inside the compiled binary,
and the GPU executed the modified instruction.

## Final thoughts

We managed to modify code running on the GPU without recompiling it. More
importantly, we have found what appears to be an instruction field containing
our constant. Given what the kernel does, it is likely part of a `mov`-like
instruction, although we haven't decoded it yet.

So the next question is obvious: what is the real encoding of the instruction?
