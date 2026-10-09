+++
title = 'Reverse Engineering NVIDIA CUDA Binaries with Data Clustering and AI'
date = 2026-10-05T19:19:52+02:00
draft = false
+++

![DBSCAN sample](/images/dbscan.png)

## Motivation

I am not a data scientist/engineer, but I wanted to do a little experiment on NVIDIA Cubin binary code. I recalled my colleague's thesis on Data Clustering and wondered what would actually happen if we tried something like that on an NVIDIA binary?

My assumption was that, if we managed to find a good distance metric, we could categorize instructions and gain a higher-level insight into how NVIDIA CUDA binaries are constructed.

## Hamming distance

The Hamming distance between two fixed-length binary words is the number of bit positions at which they differ. For our 64-bit words, we can calculate it using XOR followed by a population count:

```py
dist = (insn1 ^ insn2).bit_count()
```

The XOR identifies the differing bits, and `bit_count()` counts them. The fewer bits two words differ in, the closer they are under this metric.

Mathematically, we can view our data as a subset of the 64-bit binary words, with the Hamming distance as the metric. In other words, our two instructions are closer the more bits they have in common. They are further apart the more bits they differ in.

## Obtaining data

My own CUDA kernel code could provide very little data. It was about 24 instructions in length, and rather than adding random code, I decided to check out the `cuda-samples` repository:

https://github.com/NVIDIA/cuda-samples

My target architecture, `sm_50`, was too old to compile these samples using the documented instructions, so I simply picked one sample and compiled it manually:

```sh
cd ./cpp/5_Domain_Specific/recursiveGaussian
nvcc -arch=sm_50 -cubin recursiveGaussian_cuda.cu -I /home/stjepan/Develop/cuda-samples/Common/ -o kernel.cubin
```

This gave me the Cubin ELF, but I still needed to extract the raw binary code. I used the NVIDIA-provided tools for this:

```text
$ cuobjdump -ltext ../kernel.cubin
SASS text section 1 : kernel-_Z24d_recursiveGaussian_rgbaPjS_iiffffffff.sm_50.elf.bin
SASS text section 2 : kernel-_Z22d_simpleRecursive_rgbaPjS_iif.sm_50.elf.bin
SASS text section 3 : kernel-_Z11d_transposePjS_ii.sm_50.elf.bin
```

To extract these sections, change `-ltext` to `-xtext`. My small database of NVIDIA binary data was now much more promising (and more statistically significant):

```text
$ ls *.bin | xargs -I{} xxd -e -g 8 -c 8 {} | wc -l
2280
```

Subsequently, I compiled a lot more CUDA binaries and added more than `40000` instructions. Due to performance constraints, I opted to limit the dataset to `12000` instructions.

## First attempt (DBSCAN)

For my first experiment, I was not at all disappointed. I installed the `scikit-learn` and `numpy` packages for my Python script. Scikit-learn provides the DBSCAN clustering algorithm, and my first attempt was something as simple as:

```py
import numpy as np
from sklearn.cluster import DBSCAN
from pathlib import Path
import struct

def cluster(insns):
    dist = np.zeros((len(insns), len(insns)), dtype=np.float64)

    for i in range(len(insns)):
        for j in range(i + 1, len(insns)):
            d = (insns[i] ^ insns[j]).bit_count()
            dist[i, j] = d
            dist[j, i] = d

    clusters = DBSCAN(
        eps=5,                 # epsilon (how far points in a cluster can be)
        min_samples=10,        # minimum number of samples a cluster must have
        metric="precomputed"
    ).fit_predict(dist)

    c_dict = {}
    for i in range(0, len(clusters)):
        c = clusters[i]
        if not c in c_dict:
            c_dict[c] = [insns[i]]
        else:
            c_dict[c] += [insns[i]]

    return c_dict
```

### Sample output

This already provided useful data. The following is output showing five samples per cluster (the original dataset contains `12000` sampled instructions):

```text
_______[   0  ]_______  _______[   9  ]_______  _______[  18  ]_______  _______[  29  ]_______
  0x001cc400e22007f6      0xe2400ffff400000f      0x5c403200026720ff      0x1e03fb8aa3b7170c
  0x001fd801fec0075d      0xe2400fffe408000f      0x4c413008001722ff      0x1e03fb8aa3b71a08
  0x089fec02fc2007f1      0xe2400fffff87000f      0x4c413008001723ff      0x1e03fb8aa3b7120b
  0x001ff400ffa007f0      0xe2400fffff87000f      0x5c423200017714ff      0x1e03fb8aa3b7190a
  0x003fd800e3a007fd      0xe2400fffc889000f      0x5c423200019718ff      0x1e03fb8aa3b7140c
_______[   1  ]_______  _______[  32  ]_______  _______[  16  ]_______  _______[  30  ]_______
  0x4c98078000870001      0x32807fdf80070303      0x30cd83ff800722ff      0x37be03c2fc070c1f
  0x4cb8000005472a03      0x32807fdf80082222      0x30cd83ff80070eff      0x37be03c2fc070817
  0xf0c8000002570002      0x32807fdf80092323      0x30cd83ff800704ff      0x37be03c2fc070b07
  0xf0c8000002170004      0x32807fdf80080e0e      0x30cd83ff80070dff      0x37be03c2fc070a0f
  0x1c00180000070300      0x32807fdf80091010      0x30cd83ff800707ff      0x37be03c2fc070c27
_______[   2  ]_______  _______[  10  ]_______  _______[  17  ]_______  _______[  38  ]_______
  0x0407f80000070000      0x040007fffff7030a      0x36bd83ff80072397      0x5bdf7f81f0670506
  0x0423f80000070a05      0x040007fffff70806      0x36bd83ff80072287      0x5bdf7f81c0770705
  0x0427f80000072121      0x040007fffff72020      0x36bd83ff8007108f      0x5bdf7f81c0470404
  0x0427f80000072121      0x040007fffff70a0c      0x36bd83ff80070e87      0x5bdf7f81c0770705
  0x0407f80000070404      0x040007fffff70604      0x36bd83ff80070597      0x5bdf7f81c0470404
_______[   3  ]_______  _______[  36  ]_______  _______[  19  ]_______  _______[  33  ]_______
  0x4b68038800070007      0x38423000002709ff      0x3659038000a71514      0x537104080007060a
  0x4b6d038005470007      0x38403000001709ff      0x3659038000a70517      0x537104080007060a
  0x5b6403800ff70317      0x38423000002709ff      0x3659038000aa1617      0x5b710a000087060c
  0x5b20030800970605      0x38403000001709ff      0x3659038000a71618      0x5371020800070608
  0x5b28038000970607      0x38403000007700ff      0x3659038000a71519      0x5371020800070616
_______[   4  ]_______  _______[  11  ]_______  _______[  20  ]_______  _______[  34  ]_______
  0xe260000052000040      0xda1005affff70202      0x36b403c080071317      0x386800437f070404
  0xe29000002a000000      0xda1005affff70404      0x36b403c080071307      0x386800437f070505
  0xe290000020000000      0xda1005affff70202      0x36b403c080071707      0x386800437f070606
  0xe2600000fc800040      0xda1005affff70404      0x36b403c080071707      0x386800437f070707
  0xe290000015000000      0xda0006effff70303      0x36b403c080071707      0x386800437f070206
_______[   7  ]_______  _______[  12  ]_______  _______[  21  ]_______  _______[  35  ]_______
  0x1c00ffffffe70506      0x3868003f00070a12      0x328002c000071414      0x5be70b0780d70c0c
  0x1c0ffffffff70606      0x3868003f00070f0f      0x328002c000071313      0x5be7098780e70c0c
  0x1c0ffffffff7050d      0x3868003f0007060a      0x328002c000071313      0x5be7070780f70c0f
  0x1c0ffffffff70003      0x3868003f00070709      0x328002c000071313      0x5be7068780870f08
  0x1c0ffffffff70603      0x3858003f00070000      0x328002c000071313      0x5be70b0780970808
_______[  28  ]_______  _______[  13  ]_______  _______[  22  ]_______  _______[  37  ]_______
  0x5c4707000047ff04      0x0103f8000007f007      0x368403c010071e17      0x1e03f3504f370303
  0x5c6007800027ff03      0x0103f8000007f018      0x368403c010071e07      0x1e03f3504f370b0b
  0x5c2107800037ff03      0x0103f8000007ff03      0x368403c010071e07      0x1e03f3504f370e0e
  0x5c60078001a7ff17      0x0103f8000007f021      0x368403c010071e07      0x1e03f3504f370d0d
  0x5c6007800127ff09      0x0103f8000007f027      0x368403c010071e07      0x1e03f3504f370d0d
_______[  -1  ]_______  _______[  14  ]_______  _______[  23  ]_______  _______[  39  ]_______
  0x4cc0018005470404      0x4801038006670310      0x002c4800eee00711      0x36b403c37f070a17
  0x5cc002a000870705      0x4801038006670414      0x002c4800eee00711      0x36b403c37f071f07
  0x5b28030000470505      0x4801038006670515      0x002c4800eee00711      0x36b403c37f070b1f
  0x5c4707000032ff05      0x4801038006670615      0x002c4800eee00711      0x36b403c37f07090f
  0x32807fdf80070202      0x480103800667070e      0x002c4800eee00711      0x36b403c37f072607
_______[   5  ]_______  _______[  24  ]_______  _______[  25  ]_______  _______[  40  ]_______
  0x366c03800037050f      0x36f0058010171410      0x011c7801e2404776      0x5bae05000ff7200a
  0x3668038000170407      0x3670054000071416      0x011c7801e2404776      0x5bae0f800ff70808
  0x366803800fd72707      0x3670054000071416      0x011c7801e2404776      0x5bae05800ff7200b
  0x366820000fd72407      0x3670054000071416      0x011c7801e2404776      0x5bae04800ff71f09
  0x366c03800fe72007      0x3670054000071416      0x011c7801e2404776      0x5bae13000ff70808
_______[   6  ]_______  _______[  15  ]_______  _______[  26  ]_______  _______[  41  ]_______
  0xeed4200000070407      0xf0a81b8000070000      0x003ff401e3a00f06      0x338001437f070302
  0xeed4200000070408      0xf0a81b8000070000      0x003ff401e3a00f06      0x338003437f070c0a
  0xeedc200000070407      0xf0a81b8000070000      0x003ff401e3a00f06      0x338003437f070006
  0xeed4200000070805      0xf0a81b8000070000      0x003ff401e3a00f06      0x338003c37f070006
  0xeed4200000070c0b      0xf0a81b8000070000      0x003ff401e3a00f06      0x338003437f070006
_______[   8  ]_______  _______[  31  ]_______  _______[  27  ]_______  _______[  42  ]_______
  0xf0f0000034170000      0x30cc03ff80072326      0xd820056ff0470606      0x010437f00007f025
  0xf0f0000034670000      0x30cc03ff80070e0c      0xd82005aff0570b08      0x010437f00007f025
  0xf0f0000034470000      0x30cc03ff80071011      0xd82005aff0570609      0x010437f00007f025
  0xf0f0000034270000      0x30cc03ff80070406      0xd82005aff057070d      0x010437f00007f025
  0xf0f0000034670000      0x30cc03ff80070507      0xd82005aff0570a0e      0x010437f00007f025
```

DBSCAN has produced groups of binary words with similar bit patterns. The only exception is the `-1` cluster, which essentially means "unassigned" or, as I prefer to call it, "noise". DBSCAN puts data there when it cannot assign it to a cluster.

### Sorting things out further

In clusters `13` and `42`, we can see the MOV-like instruction from our [previous](../nvidia-reverse-engineer-gpu-code2) reverse-engineering exercise:

```text
_______[  13  ]_______  _______[  42  ]_______
  0x0103f8000007f007      0x010437f00007f025
  0x0103f8000007f018      0x010437f00007f025
  0x0103f8000007f01d      0x010437f00007f025
  0x0103f8000007f021      0x010437f00007f025
  0x0103f8000007f027      0x010437f00007f025
```

Similarly, here is the second instruction we tinkered with in the previous article:

```text
_______[   6  ]_______
  0xeed4200000070407
  0xeed4200000070408
  0xeedc200000070407
  0xeed4200000070805
  0xeed4200000070c0b
```

How could we sort these instructions out further? One thing we can do is apply bitwise AND to instructions like the ones in bins `13` and `42` to find out what they have in common:

```py
TEST = [ 0x0103b8080817f003,
         0x0103f8000007f002,
         0x0103b8080817f000,
         0x0103f8000007f006,
         0x0103b8080817f000,
         0x010437f00007f025 ]

and_val = TEST[0]
for each in TEST:
    and_val &= each

print(f"{and_val:016x}")
```

For example, this gives us the following:

```text
010030000007f000
```

So, except for the digit `3`, we seem to be getting closer to the instruction's structure. If we could combine this with our clustering approach, we might be able to extract more information about the format.

## Second attempt (adaptive HDBSCAN)

A little investigation led me to HDBSCAN, which seemed to give me more control over the clustering results:

```py
clusters = hdbscan.HDBSCAN(
               min_cluster_size=min_size,
               min_samples=samples,
               cluster_selection_method="eom",
               metric="precomputed"
           ).fit_predict(dist)
```

It seemed to produce more specific clusters, but many samples still ended up in the "noise" cluster.

### Sorting instructions

I had already described how we can use bitwise AND to get the common bit pattern of a cluster. Bitwise OR can also provide useful information about which bits vary across the data. My code combined these two operations while preserving cluster-size information:

```py
c_dict = {}
noise = []
for i in range(0, len(clusters)):
    c = clusters[i]
    if c == -1:
        noise.append(insns[i])
    elif c not in c_dict:
        c_dict[c] = [insns[i]]
    else:
        c_dict[c] += [insns[i]]

res = {}
for (key, val) in c_dict.items():
    first = val[0]
    orval = val[0]
    for each in val[1:]:
        first &= each
        orval |= each
    orval ^= first
    if first not in res:
        res[first] = (orval, val)
    else:
        (old_orval, old_val) = res[first]
        res[first] = (old_orval | orval, old_val + val)
```

Later, I also added per-bit frequencies and a noise percentage:

```py
total = 0
for (key, val) in res.items():
    total += len(val[1])
    print(f"{key:016x} {val[0]:016x} ({len(val[1]):4})")
    freqs = bit_frequencies(val[1])
    print_frequencies(freqs)
    count = 0
    for each in val[1]:
        if count > 5:
            break
        else:
            print(f"    {each:016x}")
            count += 1
    print("")

print(f"TOTAL: {total}")
noise_len = len(noise)
print(f"NOISE: {noise_len:4} {(noise_len/(total + noise_len)) * 100}%")
```

The results were OK, but not entirely satisfying. Here is one example:

```text
5c9807800ff00000 00000000000f003f (  85)
 * 01011100100110000000011110000000000011111111____00000000000_____
    5c9807800ff7001e
    5c9807800ff7001b
    5c9807800ff70017
    5c9807800ff7000b
    5c9807800ff80020
    5c9807800ff00020

308c03ff80070000 0671800000003fff (  45)
 * 00110__01___110__000001111111111100000000000011100______________
    30cc03ff80072220
    30cc03ff80072326
    30cd83ff800722ff
    36bd83ff80072397
    36bd83ff80072287
    30cc03ff80070e0c

e34000000007000f 0000000000000000 ( 125)
 * 1110001101000000000000000000000000000000000001110000000000001111
    e34000000007000f
    e34000000007000f
    e34000000007000f
    e34000000007000f
    e34000000007000f
    e34000000007000f

3280004000070000 0102078140001f1f (  75)
 * 0011001_100000_000000____100000_0_00000000000111000_____000_____
    328002c000071414
    328002c000071313
    328002c000071313
    328002c000071313
    328002c000071313
    328002c000071313

5c47000000000000 0000060001f70f0f (  90)
 * 010111000100011100000100000000000000000_____0___00000___00000___
    5c47040000a70505
    5c47040000870202
    5c47040000870b0b
    5c47040000870f0f
    5c47000000c70e0e
    5c47000000470505

4880000800070000 05201f8000303f3f (  57)
 * 01001_0_10_0000000000____00010000000000000__011100______00______
    4980060800270909
    4980040800270b0b
    4ca004080027140c
    4ca0008800070f09
    4ca0008800070b0e
    4ca0000800070b0f

300000427c070000 0ffe1f8183003f3f ( 164)
 * 0011___________0000______100001__11111__0000011100______000_____
    37be03c2fc070c1f
    37be03c2fc070817
    37be03c2fc070b07
    37be03c2fc070a0f
    37be03c2fc070c27
    37be03c2fc071437

TOTAL: 6921
NOISE: 5079 42.325%
```

The bit fields indicate the per-bit frequencies. If the frequency of a particular bit in the cluster exceeds or falls below a threshold, it is represented as `1` or `0`, respectively. If it varies across the cluster, it is represented as `_`.

The biggest problem, however, was the amount of noise. I could reduce it, but then cluster `0` grew:

```text
0000000000000000 7ffbbf9017ff3f3f (1406)
 * 010_1100_____000000000000000000000000_0_____0111000_____000_____
    4cb8000005472a03
    1c00180000070300
    4e00020000270204
    5cb8010000370a05
    5cb0118000670a06
    5c10000000370004
```

Many instructions ended up in this cluster because all bits had equal weights. Bits representing instruction structure could be outweighed by immediate values or addresses. I spent a lot of time tinkering with DBSCAN and HDBSCAN parameters, trying to find a better representation, but eventually I got tired of it.

## Employing the AI agent

I was concerned that the AI agent's existing knowledge of NVIDIA and Maxwell might influence an experiment intended to help me understand the binary format. I wanted to compare its conclusions with existing results, so I instructed the agent to approach the data without relying on NVIDIA-specific explanations.

I didn't ask the agent to decipher the binary independently, either. I still wanted to explore my clustering idea. So I left the agent running overnight, asking it to experiment with DBSCAN and HDBSCAN parameters, document any insights, and reduce the noise as much as possible.

I also instructed it to optimize my Python code if necessary and to try weighted Hamming distance.

### Minimizing noise

The AI agent's report on unweighted Hamming distance suggested that the clustering approach was not necessarily being used incorrectly; rather, some of the data was too dispersed to form useful clusters:

```text
| data        | best                         | clusters | noise |
|-------------|------------------------------|----------|-------|
| all words   | HDBSCAN `mcs=5, ms=1, eom`    | 572      | 28 %  |
| all words   | DBSCAN `eps=2, ms=2`          | 1886     | 14 %  |
| type A only | DBSCAN `eps=1, ms=2`          | 407      | 12 %  |
| type B only | HDBSCAN `mcs=3, ms=1, eom`    | 1318     | 27 %  |
| type B only | DBSCAN `eps=2, ms=3`          | 789      | 25 %  |

With equal weights, both algorithms performed best at relatively fine-grained settings in the configurations tested. High-entropy operand bits appeared to dominate the distance and break potential groups apart. Larger `min_samples` or `min_cluster_size` values tended to push more words into noise. HDBSCAN `eom` performed better than `leaf` in these tests, and `min_samples=1` gave the best results in the configurations compared.
```

### The big breakthrough

One finding from the AI agent immediately caught my attention: I had overlooked the possibility that word position might matter.

```text
All file sizes are multiples of 32 bytes. A comparison of words at offset 0 mod 4 (type A) with words at offsets 1 and 3 mod 4 (type B) showed different statistics: about 3,258 versus 5,300 distinct values per slot, and mean popcounts of 24.8 versus 18.6.
```

I had assumed that the binary contained only real instructions, rather than potentially including periodic non-instruction data. I checked the observation against the data:

```sh
$ xxd -e -g8 -c8 kernel-_Z9dwtHaar1DPfS_S_jji.sm_50.elf.bin | awk '{ if (NR % 4 == 1) { print $2 } }' | head -n20
001c4400fe0007f6
081fc401fec0073f
001f8400fec217f6
081fc400fea207f1
001fc000fec007f1
0002c440fe0007b2
081fcc00fe2007e1
003fd440fe2007f2
081fc400fea007e1
041fc400fee007f6
001fd400ffe007f1
001ffc00fcc00711
001fbc00feaa0ff1
001ffc00fe0007e1
0003d000fe0007f5
001fbc00fde007f1
001fd400ffe007e9
001fbc00fde007ef
083fc400ffa007e8
001fc400fe2007f1
```

The words at offsets divisible by four appeared to have a different pattern from the words at offsets 1 and 3 modulo four. The AI agent's analysis helped me notice this difference, which I then checked against the data. I haven't established what, if anything, the pattern tells us about the underlying binary format.

## Final thoughts

This was my first experiment with data clustering for reverse-engineering. Although the clustering results were imperfect, the experiment showed me that Hamming distance can reveal structure in binary data.

The most useful outcome was an observation I had overlooked: words at different positions in the binary appeared to have different statistical properties. An AI agent helped me notice this pattern, which I then checked against the data.

I haven't established what this positional pattern means yet. For now, it's simply an observation worth investigating further.

An AI agent can be useful even when it doesn't solve the problem outright: it can run experiments, compare results, and help identify patterns that deserve a closer look.
