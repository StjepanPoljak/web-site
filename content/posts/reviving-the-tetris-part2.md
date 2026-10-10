+++
title = 'Summertime fun: Reviving the Tetris, Part II'
date = 2026-10-10T13:52:03+02:00
draft = false
+++

<script src="https://asciinema.org/a/kUsbEpFopGwdiypI.js" id="asciicast-kUsbEpFopGwdiypI" async></script>

## Recap

My [previous](../reviving-the-tetris-part1) article focused on explaining my
15-year-old experimental Tetris device, which was based around a PIC16F876 MCU,
an ST7920 display, and a Philips remote controller. Since my original background
is in mathematics, I actually consider this one of the most complex projects I
have ever tackled.

Now that the hardware is barely working (if at all), I have another goal: revive
the Tetris device in a software PIC emulator.

## Beginnings

It took me a while to recall the PIC16 ISA, how the MCU worked, and what its
components were, but it was also a fun summertime project. I spent a lot of time
meditatively going through the documentation, rewriting instruction decodings in
Rust, and tinkering with various PIC quirks.

## Test harness

However, although simply rewriting the instructions from the documentation was
easy and rather fun in a meditative sort of way, this was a huge arena of
potential bugs. At that time, I was still not using AI agents directly in my
codebase (and I wanted to do this part myself: I was reliving the past, albeit
in a new environment).

Instead of relying on an AI agent to be more accurate in ISA reconstruction and
testing, I created a test harness in Rust that took the existing PIC emulator,
`gpsim`, as a source of truth.

### Basic test infrastructure

So, this function is the core of my test framework:

```rs
fn run_gpsim(test_case: &TestCase, gpsim_data: &GPSimData) -> io::Result<TestRegs> {
    let comm_source = test_case
        .regs
        .iter()
        .fold("break e endl\nrun\ndump".to_string(),
              | acc, x | format!("{}\nreg({})", acc, x));

    let _ = std::fs::write(&gpsim_data.comm, format!("{}\nquit", comm_source).to_string());

    print_file(&gpsim_data.comm)?;

    let gpsim_out = std::process::Command::new("gpsim")
        .args(["-c", gpsim_data.comm.to_str().unwrap(), gpsim_data.cod.to_str().unwrap()]).output()?;

    let gpsim_dump = String::from_utf8_lossy(&gpsim_out.stdout).to_string();
    let mut reg_map = HashMap::<u16, u8>::new();
    for reg in &test_case.regs {
        reg_map.insert(*reg, get_reg_at(*reg, &gpsim_dump)? as u8);
    }

    let break_pc = get_reg("pc", &gpsim_dump)?;

    Ok(TestRegs { w: get_reg("W", &gpsim_dump)? as u8,
                  pc: break_pc,
                  status: get_status(&gpsim_dump)? as u8,
                  endl: break_pc,
                  regs: reg_map })
}
```

This simple function runs `gpsim` and extracts register and memory data as
requested. It's the basis for the `run_test` function, which runs my PIC16F876
emulator and compares its registers with those from `gpsim`.

This has sped up my manual work enormously (even without wasting tokens on AI
agents).

### Test example

I constructed an umbrella test function for various PIC16F876 instruction
categories:

```rs
fn test_wf(insn: &str) {
    for _ in 0..TESTS_PER_INSN {
        let op1 : u8 = rand::random_range(0..255);
        let op2 : u8 = rand::random_range(0..255);
        let addr : u16 = rand::random_range(0x20..0x7f);
        let wf : u8 = rand::random_range(0..1);
        run_test(&TestCase { name: insn.to_string(),
                             source: get_wf_src(insn, op1, op2, addr, wf),
                             regs: vec![ addr ] },
                 |hex| assert!(!hex.is_empty())).unwrap();
    }
}
```

This function could then be reused, as shown in the examples below:

```rs
#[test]
fn test_addwf() {
    test_wf("ADDWF");
}

#[test]
fn test_subwf() {
    test_wf("SUBWF");
}

#[test]
fn test_andwf() {
    test_wf("ANDWF");
}
```

## Getting to first Tetris screen

The basic infrastructure for my project in the emulator was simply:

```text
           PIC16F876       ST7920         Linux
           emulation     emulation       terminal
            ______                        ______
           -|    |-  -->   010010   --->  |>   |
           -|    |-  -->   001101   --->  |____|
    /--->  -|    |-
   /  /->  -|____|-    abstract keypad      | |
   | |                    emulation         | |
   | |                                      | |
   | \------------------   100101   <------/  |
    \-------------------   001011   <--------/
```

By using my test infrastructure and manually tracing PIC instructions, I was
soon able to get to the welcome screen on my Tetris console.

## Enter AI agent

However, keyboard input did not work. I debugged the pipeline and confirmed that
the input reached the PIC16F876, but the code was still in the polling loop. It
seemed like everything was running perfectly, but it obviously wasn't. Soon the
summer was over, and I needed to focus on my other projects (KVM in Rust VMM, my
own OS, technical writing, and CUDA reverse-engineering), so eventually I let
the project sit and wait.

A few months later, having been busy with other stuff, I had already started
utilizing AI agents to get a grasp on all the unfinished work I had. Last night,
I simply left the agent working overnight with instructions on how to build the
project, run it, and debug it. In the morning, I had my old Tetris device
running in the Linux terminal.

## Conclusion

Fifteen years ago, I created an embedded Tetris device. I revived it a few
months ago in a software emulator. I utilized a specific test harness to test it
against existing emulation software. This sped up my development enormously.

Now we are in the age of AI, where reconstructing a forgotten old project is
just a prompt away. I don't doubt that AI could have reconstructed the whole
project from scratch — it's just that I wanted to tinker with low-level details.

Reading documentation and copying an ISA no longer makes sense to me. I have a
plan to expand and improve this project using only AI agents from now on, so I
can focus on more complex stuff. The project will grow into an arena for
experimentation with various architectures.

Links to the emulator and Tetris device repositories:

https://github.com/StjepanPoljak/rupic

https://github.com/StjepanPoljak/TetrisDevice/
