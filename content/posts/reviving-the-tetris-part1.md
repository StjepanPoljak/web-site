+++
title = 'Summertime fun: Reviving the Tetris, Part I'
date = 2026-09-11T17:38:34+02:00
draft = false
+++

{{< youtube TQ3UTt3WZ6Y >}}

## Backstory

It started as a joke on a summer fifteen years ago.

I was in my last year of college (Department of Mathematics) and I jokingly told a colleague that he should buy me a Nintendo DS for my birthday.

Unfortunately, he did not think that was a good idea. So I told him that I will
make one myself.

And the idea stuck with me.

## First steps

My father was an electric engineer and had tons of old electric components from
the 1980s Yugoslavia (he worked in Belgrade before the war). So, naturally, I
started using my father's stash to create my first circuits on an old
breadboard.

I was never good with physics and so in one of my experiments a capacitor
started to weirdly deform. I leaned to take a closer look and just at that time
the capacitor exploded and flew past my head like a bullet. It was a really
close call and a lesson learned the hard way.

I also started experimenting with infrared sensors and simple PIC MCUs,
especially the P12 architecture. At the time, I spent a considerable amount of
money on the only PIC programmer I could find locally, which was, of course,
already outdated.

### Hardware prototype

The first hardware prototype ended up in the trash. I was still learning to
solder and it showed.

The second prototype turned out much better. I bought a more capable PIC MCU
for it, the PIC16F876, and designed the board so it could be expanded later. I
left pin headers connected to unused PIC pins, allowing additional components
to be added as the project evolved.

<p align="center">
  <img src="https://github.com/StjepanPoljak/TetrisDevice/blob/master/TetrisFoto.jpg?raw=true" alt="The original Tetris console">
</p>

## Programming

Once I got the hardware working, the real programming began. And it was an unforgettable experience. Something to be told to my grandchildren in a "we had
to physically walk the room to compile a small binary" story.

The programmer I had only supported the old COM1 port and my PC was already too
modern for it. My parents, however, had a PC with COM1 port in the kitchen, but
it was too old to run the software I needed.

So my development workflow became:

1. Write and compile the code on my PC.
2. Copy it to a USB stick.
3. Physically walk to the kitchen.
4. Upload it to the PIC.
5. Walk back.
6. Discover that something was wrong.
7. Repeat.

Eventually, I bought the only display I could find locally: an ST7920. And now
I had something that could actually display the game.

### Tetris logic

Implementing Tetris on an 8-bit PIC presented some interesting constraints.

I needed a 10-bit representation for the playing field, but the PIC was an
8-bit MCU.

Moving pieces left and right therefore required shift operations across a
wider-than-native integer. I created a small wrapper around the shift
operations so that they behaved as if they were operating on 10-bit values.

Rotation was handled differently. Instead of calculating rotations at runtime,
I stored the different orientations of each piece in memory and simply selected
the appropriate representation.

## Display

Getting the display working turned out to be one of the more difficult parts.

And even after I got it working, the result was... small and unsatisfying.

The native drawing operations produced a very small image, so I added another
layer of wrappers around the drawing functions that effectively quadrupled the
pixels.

That made the game considerably easier to see - and much more usable.

## Controls

One of my earlier experiments with infrared sensors unexpectedly provided the
solution for the controls. I had figured out how to decode the byte stream coming from a Philips controller so I simply reused it.

{{< youtube ZJMdeUvckCg >}}

The controller became the input device for my Tetris machine, while a separate
P12 MCU handled the incoming signals and connected to the remaining pins of the
main system.

<p align="center">
  <img src="https://github.com/StjepanPoljak/TetrisDevice/blob/master/ReceiverFoto.jpg?raw=true" alt="PIC12 Remote Module">
</p>

## The result

The end result was a fully playable physical Tetris game as can be seen in the
really low-resolution video below.

{{< youtube wAj_gRQC4ok >}}

Its future, however, remained uncertain.

From time to time, something would stop working. I would have to instrument the
hardware, resolder a few wires, or sometimes just wiggle them until the machine
decided to cooperate again.

Eventually, the console stopped working altogether.

By then, I no longer had the proper programmer or debugging equipment, so
bringing it back to life wasn't particularly practical.

But two things survived: the code and the schematics. They can be found on the
following GitHub page:

https://github.com/StjepanPoljak/TetrisDevice/

And fifteen years later, I had an idea: instead of trying to repair the
original hardware, why not revive the project by running the original code in
an emulator?

So that's what I was doing this summer.
