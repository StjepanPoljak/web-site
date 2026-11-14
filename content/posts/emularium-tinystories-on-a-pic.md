+++
title = 'A Language Model on a PIC16: Emularium Tells Stories'
date = 2026-11-14T07:00:00+01:00
draft = true
+++

![TinyStories on the emulated LCD](/images/emularium-tinystories-lcd.png)

## Recap

In [Reviving the Tetris, Part II](../reviving-the-tetris-part2) I wrote
that my PIC16F876 emulator would grow into an arena for experimenting with
computer architectures, and that from then on I would extend it only
through AI agents. The emulator has a new name,
[Emularium](../../projects/emularium/), and it now emulates whole boards:
a board is a KiCad schematic or a small TOML file, and Emularium wires up
the parts, loads the firmware and runs everything in real time, with the
displays served over VNC.

This post is about the first real experiment: making the same 8-bit
microcontroller that ran my Tetris write children's stories with a
language model.

## The idea

Andrej Karpathy trained a family of tiny Llama-style models on the
TinyStories dataset. The smallest, **stories260K**, has 260 thousand
parameters: 5 layers, a model width of 64, 8 attention heads sharing 4
key/value heads, and a vocabulary of 512 tokens. It writes simple
children's stories.

The PIC16F876 has:

- 368 bytes of RAM,
- 8K words of program flash,
- no hardware multiplier,
- 5 million instructions per second at 20 MHz.

The weights alone are about 270 KB, so they have to live outside the chip,
and every token means reading all of them once.

## Step 1: an integer model as the specification

Before touching any assembly, the model was rebuilt in Python with only the
arithmetic a PIC can do:

- weights as int8 with one power-of-two exponent per row,
- activations in *block floating point*: int8 mantissas plus one exponent
  per vector,
- every rescale a rounding shift,
- every non-linear function (1/sqrt, exp2, sigmoid, sine) a small lookup
  table.

On 100 validation stories this integer model has a perplexity of 3.420,
against 3.387 for the original floating-point model, about 1% worse. Its
top token agrees with the float model's 95% of the time.

The important part is that this Python model is **bit-exact**: it defines
every intermediate value. A script dumps all of them (residual stream,
normalised activations, q, k, v, the KV cache entries, attention output,
FFN hidden vector, logits, sampled token) for every position and layer.
That trace became the test oracle for the firmware.

## Step 2: the board

![The TinyStories board, drawn from its TOML file](/images/emularium-tinystories-board.svg)

- **U1**, a PIC16F876 at 20 MHz.
- **U2-U4**, three Microchip 25LC1024 SPI EEPROMs (128 KB each) holding the
  weight image.
- **U5**, a 23LC1024 SPI SRAM for the KV cache and scratch space.
- **J1**, the same ST7920 128x64 LCD as the Tetris.

The diagram above is generated from the board's TOML file by a new
`tools/schematic.py`, which draws parts as boxes and groups the nets: the
SPI bus is a single hub, and every line lists the pins it carries.

Emularium had no SPI EEPROM, so the 25LC1024 got a model of its own: the
instruction set, write enable, 256-byte page writes, the status register
with block protection, the self-timed write and erase cycles, and deep
power-down. It was checked against the datasheet (DS22064C), which caught
a few details: the release from deep power-down after only the RDID
opcode, the 100 µs before the chip answers again, and the 25 ns output
disable time. As with the SPI RAM before it, the traffic the PIC produces
is also decoded by **sigrok**'s SPI decoder and compared with what the
datasheet says should be on the bus, so the check doesn't depend on the
emulator being right.

## Step 3: the firmware

One token is a long trip:

1. **Embedding**: the token's row from the EEPROM.
2. **Five layers**, with the weights read as one SPI stream across the
   three EEPROMs:
   - RMSNorm: a 40-bit sum of squares and `1/sqrt` from a table,
   - q, k and v: matrix rows of offset-binary bytes,
   - rotary position embedding from a quarter-wave sine table,
   - the new k and v into the KV cache in the SRAM,
   - attention for each of the 8 heads over every cached position,
   - the output projection and the residual,
   - the feed-forward network (SiLU) and the second residual.
3. **Classifier**: 512 logits into the SRAM.
4. **Sampling**: the next token, with a 32-bit xorshift random number.
5. **Printing**: the token's text on the LCD, word-wrapped and scrolling.

A few tricks make this fit:

- **Unsigned multiplies only.** Weights and activations are stored offset
  by 128, and `sum(a*w) = sum(ua*uw) - 128*(S + A)`, with `S` stored in each
  row's header and `A` computed once per vector. The inner loop multiplies
  two unsigned bytes into a 24-bit accumulator in 47 cycles, not counting
  the SPI read; the multiply even starts from the accumulator's low byte,
  so the addition is free.
- **Measure, then size.** Every intermediate's range was measured over a
  few hundred positions before choosing its width: matrix accumulators fit
  in 24 bits, normalisation products need 32, and only one product in
  attention needs 64.
- **The SRAM as scratch.** The residual stream (128 bytes) and the
  activations (up to 172 bytes) take most of the RAM. The attention
  output, the FFN's 172 hidden values and the 512 logits go to the SRAM
  instead.
- **Two-pass attention.** The softmax needs the maximum score first, so
  each head reads its cache twice: once for the maximum, once for the
  weights and the weighted sum. The result is scaled by
  `round(2^32 / sum)` instead of dividing per element.
- **Two flash pages.** The code (3.6K words) doesn't fit in one 2K page.
  Page 0 holds everything the inner loops call, page 1 the rest, and a
  small script checks every `CALL` and `GOTO` in the listing against the
  page bits it will run with.

## Step 4: testing against the trace

The firmware has labelled checkpoints. The test assembles it, runs it on
the emulated board with the real weights, stops at each checkpoint and
compares the PIC's RAM and the SRAM with the Python trace: the
embedding, then for every layer the normalised activations, q/k/v, the
cache entry, the attention output, the residual, the FFN, and finally the
logits and the sampled token.

That made the bugs easy to find. Three examples:

- **RoPE was wrong, but only in the odd elements.** A helper that loads a
  24-bit value left the pointer on its last byte, so the second value of
  each pair was read one byte early.
- **Attention averaged over 257 positions at position 0.** `CLRF` sets the
  zero flag, and a carry test came right after one, so the context length
  was always "256 + 1".
- **A flash-page check that checked nothing.** The configuration word at
  address `0x2007` comes early in the listing, and the checker stopped at
  the first address above the code.

The last bug showed up 88 tokens into a story: the sigmoid table stores
its values minus 128, and adding 128 back turned the largest value, 256,
into 0. With that fixed, every value of every layer matches the Python
reference.

## The result

At 20 MHz a token takes about 6 seconds: roughly 259 thousand
multiply-accumulates, each with a bit-banged SPI read, plus attention
over the growing context. The emulator runs it in about real time, so a
story builds up word by word in the VNC window over a few minutes:

> Once upon a time, there was a little girl named Lily. She loved to play
> in the kitchen. One day, she found a new bear who looked like a bee. She
> drove around at the beach and said, "Hello there! Do you like things?"

It's charmingly incoherent, as a model with 260 thousand parameters
should be. But every word of it was computed by an 8-bit microcontroller
designed in the 1990s, with 368 bytes of RAM.

## Related work

stories260K has been run on 32-bit microcontrollers many times, for
example on the [ESP32-S3](https://github.com/Riteshbstkr/esp32-llm-n16r8)
and the [STM32H7](https://github.com/schuhandreas/embedllama-stm32h7a3),
which have hundreds of kilobytes of RAM. On 8-bit machines, the closest
projects I found are:

- [ternary.c64](https://github.com/demartinogiuseppe/ternary.c64): a
  Llama-style model with 688K ternary weights on a Commodore 64,
  bit-exact against a C reference, so far only in an emulator;
- [Soul Player C64](https://github.com/gizmo64k/soulplayer-c64): a
  25K-parameter transformer on a Commodore 64;
- [bitnet-6502](https://mattbeton.com/blog/bitnet-6502.html): a ternary
  recurrent model running on a real BBC Micro.

Emularium's chip is smaller: 368 bytes of RAM against the 6502 machines'
32-64 KB. It also runs Karpathy's original trained model, quantized after
training, rather than one trained for ternary weights. I couldn't find
another language model running on a PIC16.

## How it was built

True to my plan, I didn't write this code. I wrote the goal and the
constraints, and Claude, an AI agent, did the rest over two nights:

- the integer model, the measurements and the roadmap,
- the EEPROM model and its datasheet check,
- about 3,000 lines of PIC assembly,
- the tests and the board diagrams.

My part was reviewing: the decisions, the commits, and this article.

What made it work is the same thing that made the Tetris emulator work: a
source of truth. For the CPU it was `gpsim`, for the SPI traffic `sigrok`,
and for the model the bit-exact Python trace. With a good oracle, an
agent can be trusted with 3,000 lines of assembly.

## What's next

- **Hardware SPI.** The PIC's MSSP shifts a byte in 8 cycles instead of
  about 35, which would hide the weight reads behind the multiplies:
  roughly 1.7 times faster.
- **Four PICs in a ring.** Each owns one key/value head and a quarter of
  the FFN, and they exchange partial sums over their serial ports. The
  Python model of this is bit-identical to the single PIC, at about 0.7 s
  per token.
- **Real hardware**, from the KiCad schematic.

The code is on GitHub:

https://github.com/StjepanPoljak/Emularium
