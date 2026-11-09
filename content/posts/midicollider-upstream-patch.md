+++
title = 'How MIDICollider Led to an Upstream Linux Kernel Patch'
date = 2026-11-04T17:20:49+01:00
+++

![MIDICollider in Linux](/images/midicollider-kernel.png)

## Introduction

MIDICollider is a Raspberry Pi 3B+ that works as a standalone SuperCollider
instrument and receives input from a USB MIDI keyboard. Apart from the Linux
kernel there's a custom PID 1 (`init`) and three processes: `scsynth`, `midid`
(raw MIDI to OSC) and `recvd` (file uploads over Ethernet). Everything the init
process and the services print, plus the kernel log, goes to `log.txt` on the
SD card's data partition, so the Pi can be debugged without a serial cable
(although with the hassle of manually transferring the SD card).

### The setup

| | |
|---|---|
| Board | Raspberry Pi 3B+ (BCM2837, 4× Cortex-A53, 64-bit) |
| Kernel | Raspberry Pi tree, 6.12.61-v8, built by Buildroot 2026.02.3, no modules |
| Keyboard | Nektar Impact LX49+, USB ID `2467:2015`, full speed, bus-powered, class compliant |
| USB path | Pi's DWC OTG controller (`dwc_otg` driver) → on-board hubs → port `1-1.1.2` |
| Consoles | HDMI (`tty1`) and serial (`ttyAMA0`, 115200 baud), kernel loglevel 7 |
| MIDI path | `/dev/snd/midiC1D0` (raw MIDI, no alsa-lib) → midid → OSC → scsynth |

The midid process already handles hotplug. It rescans `/dev/snd` every second
and opens any new `midiC*D*`. When a `read()` fails (an unplugged USB device
gives `ENODEV`), it closes the device and releases every held note.

## The bug

I had a problem on both my laptop and the Raspberry Pi: whenever I switched my
USB MIDI keyboard off and on, MIDI input stopped working. No sound.

Really bad, especially as I have two toddlers, and they just love pressing
random buttons on the keyboard.

So I turned to AI for help - and ended up writing a Linux kernel patch.

### What the log showed

The boot was normal: scsynth started, midid opened `/dev/snd/midiC1D0`, and
the piano played for hours. Then, when I switched the keyboard off and on:

```text
[42445.106445] kernel: usb 1-1.1.2: urb status -32
[42445.111233] kernel: usb 1-1.1.2: urb status -32
[42445.115942] kernel: usb 1-1.1.2: urb status -32
... 251 lines in total, one every ~4 ms ...
[42446.116049] kernel: usb 1-1.1.2: urb status -32
```

And then nothing more, ever. What's missing seemed to matter as much:

 - no `usb 1-1.1.2: USB disconnect`
 - no new device being enumerated when the keyboard came back on
 - no `midid: closed /dev/snd/midiC1D0`

First things first, `-32` is `EPIPE`: the device answers the host's requests
on its MIDI IN endpoint with a `STALL` handshake, which a keyboard does while
its power is going away.

As far as the log could tell, the keyboard never left. I concluded that the
Pi hadn't noticed it going away: switching it off didn't take it off the bus
long enough for the hub to see. That conclusion was wrong, but it took three
days to find out.

### Brainstorming

That was AI's diagnosis. Its proposed solution was to read `/dev/kmsg`, look for
the `urb` status messages, and re-enumerate the device via `/sys/bus/usb/devices/<port>/authorized.`

Well, I've been in this industry quite a while and there were two problems with
this: performance and no guarantee that the message will remain the same (or
even visible due to, e.g. `printk` ratelimiting).

To be quite frank, to me it sounded like the real fix was to be done in the
Linux kernel itself. And this led me to a more thorough investigation.

## The USB MIDI driver

USB devices never transmit on their own: the host asks and the device
answers.

1. When midid opens `/dev/snd/midiC1D0`, snd-usb-audio's `substream_open()`
   calls `snd_usbmidi_input_start()`, which submits 7 input URBs (USB transfer
   requests).
2. The USB controller keeps sending the keyboard `IN` tokens. With nothing to
   send, the keyboard answers NAK and the host asks again.
3. When a key is pressed, the keyboard answers with a 4-byte USB-MIDI event
   packet: `09 90 3C 64` is cable 0, code index 9 ("note on, 3 bytes"),
   followed by the MIDI message itself.
4. That packet completes the URB. The controller interrupts, USB core gives
   the URB back, and `snd_usbmidi_in_urb_complete()` runs, in atomic context
   with `urb->status == 0`.
5. `snd_usbmidi_standard_input()` unpacks the packet, `snd_rawmidi_receive()`
   queues `90 3C 64`, and whoever is reading the rawmidi device wakes up.
6. The handler resubmits the URB, so the host never runs out of pending reads.
7. midid's `read()` returns `90 3C 64`, `note_on()` sends `/s_new` to scsynth,
   and the note sounds.

When the keyboard powers off, step 2 goes wrong: it answers STALL, so step 4
runs with `urb->status == -EPIPE`.

### The driver's error handling

Here is the error classification in `sound/usb/midi.c`:

```c
static int snd_usbmidi_urb_error(const struct urb *urb)
{
	switch (urb->status) {
	/* manually unlinked, or device gone */
	case -ENOENT:
	case -ECONNRESET:
	case -ESHUTDOWN:
	case -ENODEV:
		return -ENODEV;
	/* errors that might occur during unplugging */
	case -EPROTO:
	case -ETIME:
	case -EILSEQ:
		return -EIO;
	default:
		dev_err(&urb->dev->dev, "urb status %d\n", urb->status);
		return 0; /* continue */
	}
}
```

So we have two classes:

- `-ENODEV`: the device is gone. Stop.
- `-EIO`: the device doesn't answer, or answers garbage, as happens while
  it's being unplugged. An error timer resubmits the URBs 100 ms later
  (`ERROR_DELAY_JIFFIES`), without a message, until the disconnect arrives.

Everything else (`-EPIPE` included) will just log and "continue", so
`snd_usbmidi_in_urb_complete()` resubmits the URB straight away.

The endpoint is still stalled, so the resubmitted URB fails again, gets
logged, gets resubmitted... That's the 250 lines a second.

## The first fix

For the first version, in `snd_usbmidi_in_urb_complete()`, `-EPIPE` I added a
100 ms timer retry instead of an immediate resubmit and queued a device reset.
The `-EIO` errors queued a reset only after 2 s of failing. A rate limit made
sure a reset that USB core dropped (it gives up if it can't take the device
lock within a second) would be asked for again.

On the Pi it seemed to work perfectly. Each off/on logged one
`MIDI input not recovering (urb status -32), resetting device`, then
`USB disconnect` 0.67 s later, and midid reopened the keyboard 3-4 s later.
Even unplugging the cable went through the reset first, which I took as proof
that `dwc_otg` missed real unplugs too. I sent the patch to the sound list on
2026-10-26.

### The review

Within hours, Sashiko, an AI that reviews patches posted to the kernel
lists, came back with two findings. I checked both against the code, and both
were right:

- **The failing streak latched on an idle device.** Only a successful URB,
  that is MIDI data, cleared the "failing" state. An idle keyboard sends
  nothing, so two harmless bus glitches hours apart counted as one streak of
  failing for hours and caused a needless reset. v2 starts a new streak after
  500 ms without errors.
- **The reset limit was per endpoint.** A device with two failing input
  endpoints could be reset twice. v2 keeps the limit per device.

The first bug needs a stray error to show itself, so I faked them with a
test-only patch: a timer unlinks a pending input URB and the completion
handler turns the resulting `-ECONNRESET` into `-EPROTO`. On the laptop, with
nobody touching the keys, v1 reset the keyboard on the second fake error,
reporting a 10240 ms "streak" that was really ten quiet seconds. v2 didn't.
I sent v2 on 2026-10-27.

### The maintainer

Takashi Iwai, the sound maintainer, had answered v1 a few hours before v2 went
out:

> This doesn't look like a problem to be worked around in the USB audio
> driver side, but better to be addressed in the USB core, IMO.

Fair enough: if the USB stack loses a disconnect, the USB stack should be
fixed. So the next question was where the disconnect was lost.

## Where was the disconnect lost?

`dwc_otg` is the Raspberry Pi's own driver for the USB controller; mainline
Linux doesn't have it. Mainline uses `dwc2` for the same hardware, and the Pi
can boot with it (`dtoverlay=dwc2,dr_mode=host` in `config.txt`). If `dwc2`
noticed the off/on, the bug was in `dwc_otg`.

**dwc2, with my patch still in the kernel:** four off/on cycles, each with
`MIDI input not recovering`, then `USB disconnect` 0.67 s later. "dwc2 loses
it too", I concluded, and drafted a report for linux-usb.

That was the second misreading. The patch resets the device on the first
`-EPIPE`, and every power-off produces one. That 0.67 s, the same every time,
was how long the reset took. The patch got there first, so this run couldn't
show whether the disconnect would have arrived without it.

**dwc2, without the patch:** each off/on gave a short burst of `-32` and then
`USB disconnect` **0.13-0.26 s** after the first error. The keyboard came
back by itself every time. So dwc2 was fine, the bug had to be in
`dwc_otg`, and I drafted an issue for the Raspberry Pi kernel instead.

To give that issue more detail, I tried turning parts of `dwc_otg`'s fast
interrupt handling off, but nothing helped.

The logs did have something in common, though: Every unpatched `dwc_otg` log,
the original included, stopped shortly after the errors began. Not even init's
own stats line, written once a minute, came afterwards, although the heartbeat
LED kept blinking (that's a kernel LED trigger). The `dwc2` log was the only
one that kept going.

## The real cause

The log loop again: each failure goes through `dev_err()`, at a loglevel
that's printed to every console. On this Pi that's the HDMI text console,
which draws and scrolls in software, and a 115200-baud serial port. Printing
is slow, which is why the loop ran at exactly one line every 4 ms.

And in this setup, all hardware interrupts were being handled by CPU 0.

So the test: unpatched kernel, normal `dwc_otg`, and `loglevel=3` on the
kernel command line. The messages still go into the kernel log, which init
copies to `log.txt`, but not to the consoles.

**dwc_otg with `loglevel=3`:** three off/on cycles, each with a burst of `-32`,
now about 8000 a second, and `USB disconnect` **0.04-0.24 s** after the first
error, just like dwc2. The keyboard came back every time, and the log carried
on normally.

The only thing that changed was where the error messages went. So it was
neither the USB core nor `dwc_otg`. The driver's own resubmit-and-print loop,
throttled by the consoles, kept the Pi so busy that the hub's disconnect
report was never handled. The log writer and scsynth stalled the same way:
one run logged `midid: lost scsynth` 2 s into the storm. Why `dwc2` got its
disconnect through the same storm, I don't know; its interrupt handling is
different, and three cycles aren't many.

## The fix

The driver already has the right treatment for errors that happen while a
device goes away: the `-EIO` class, retried every 100 ms without a message.
`-EPIPE` belongs there:

```c
	/* errors that might occur during unplugging or switching off */
	case -EPROTO:
	case -ETIME:
	case -EILSEQ:
	case -EPIPE:
		return -EIO;
```

A really halted endpoint wasn't cleared before either. It's now retried
every 100 ms instead of continuously, and the output direction gets the same
treatment, since it uses the same function.

On the Pi, with the normal console settings, each off/on now gives
`USB disconnect`, a new enumeration 3 s later and `midid: opened`, without a
single `urb status -32`. I sent it on 2026-10-29 as "ALSA: usb-audio: don't
resubmit MIDI URBs at once on -EPIPE" and asked for v2 to be dropped.

Looking back, v1 already contained this fix. It sent `-EPIPE` to the 100 ms
timer retry "to end the log spam", and then reset the device on top. The
retry stopped the storm and the disconnect came through; the reset only
raced it. All of the reset logic, the rate limit, the per-device fields, the
streak-latch fix and the fault injection, was solving a problem that the
first two lines had already solved.

### The laptop that went silent too

On the laptop (Ubuntu, kernel 7.0, xHCI), the keyboard reconnected cleanly
after an off/on, but Carla received nothing. That looked like the same bug, but
wasn't: PipeWire 1.6.2's ALSA sequencer bridge doesn't remove a destroyed port
from a list, and after the reconnect its data thread spun at 100 % CPU in
`process_recycle()`. The stack trace led to upstream commit `47425371`
("alsa-seq: remove port from `mix_list` on destroy"), which is in 1.6.4, but
not in Ubuntu's package. This, however, is the story for another time.

## Upstream

The patch made it into Linux 7.4-rc2, as part of the sound subsystem updates.

There were two patch series along the way:

- [First patch series — 26 October 2026](https://lore.kernel.org/linux-sound/20261026125751.1700779-1-stjepan.poljak@gmail.com/) (revised on 27 October)
- [Replacement patch — 29 October 2026](https://lore.kernel.org/linux-sound/20261029165701.2004391-1-stjepan.poljak@gmail.com/)

The second submission replaced the original recovery approach with the much
simpler fix: classify `-EPIPE` as an error that can occur during unplugging, and
let the existing retry mechanism handle it.

The final patch is titled **ALSA: usb-audio: don't resubmit MIDI URBs at once on
-EPIPE**.

## Final thoughts

The fix has now made it into Linux 7.4-rc2. What started as a minor annoyance
with a USB MIDI keyboard being switched off and on again ended up as my first
upstream Linux kernel fix.

The original patch went through two submissions. The first attempt explored a
more complicated recovery mechanism, while the second took a much simpler
approach: treat `-EPIPE` like the other errors that can occur during unplugging
and let the existing retry mechanism do its job.

Link to MIDICollider GitHub repository:

https://github.com/StjepanPoljak/MIDICollider
