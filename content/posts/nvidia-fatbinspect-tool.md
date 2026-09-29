+++
title = 'NVIDIA FatBin and Cubin Inspection with Rust'
date = 2026-09-29T17:26:03+02:00
draft = false
+++

<p align="center">
  <img src="/images/nvidia-fatbin.drawio.svg" alt="NVIDIA FatBin">
</p>

## Motivation

In my reverse-engineering NVIDIA CUDA series, I have mostly focused on
understanding and modifying the bit-level encoding of SASS instructions. Until
now, however, the tooling around those experiments has been rather... messy.
Modifying binaries without being able to reason about structures that surround
them simply makes further attempts at reverse-engineering harder to accomplish. 

## Format summary

NVIDIA CUDA host ELFs, as output by `nvcc`, contain two interesting sections,
and this is something we are able to see ourselves rather easily:

```text
$ readelf -S test | grep -i nv -A1
  [18] .nv_fatbin        PROGBITS         0000000000089cc8  00089cc8
       0000000000001288  0000000000000000   A       0     0     8
  [19] __nv_module_id    PROGBITS         000000000008af50  0008af50
       000000000000000f  0000000000000000   A       0     0     8
--
  [29] .nvFatBinSegment  PROGBITS         00000000000a8058  000a8058
       0000000000000030  0000000000000000  WA       0     0     8
```

The `.nv_fatbin` section contains pointers to FatBin headers that are located
in `.nvFatBinSegment`. FatBin headers precede the actual payload: Cubin ELFs.
Cubin ELFs contain the binaries that will be loaded and executed on the GPU.

## The fatbinspect tool

Rather than implementing another one-off parser for a particular experiment, I
decided to turn what I learned into a small Rust tool for inspecting, extracting
and modifying NVIDIA FatBins and Cubins. I called it `fatbinspect` and based the
implementation on existing research from [Stealthium](https://stealthium.io).

### Obtaining FatBin entries

The tool I wrote in Rust first uses the `object` crate to obtain the sections
named `.nv_fatbin` and `.nvFatBinSegment`:

```rs
fn main() -> io::Result<()> {
    let args = Args::parse();
    let mut data = std::fs::read(&args.binary)?;
    let elf = ElfFile64::<Endianness>::parse(&*data).unwrap();

    let fatbins: Vec<FatBin> = get_fatbin_ptrs(&elf, &data)
        .iter()
        .map(|p| get_fatbin_ptr_data(p, &data).unwrap())
        .collect::<Vec<_>>();
```

The `get_fatbin_ptrs` function reads the pointers stored in `.nv_fatbin`. Each
pointer refers to a FatBin entry in `.nvFatBinSegment`. We then follow those
pointers with `get_fatbin_ptr_data` and parse the resulting headers and payload
metadata.

#### Example

<script src="https://asciinema.org/a/rigIKrknGACpY99I.js" id="asciicast-rigIKrknGACpY99I" async></script>

### Getting Cubins

Now when we have got all FatBin entries, we can extract their payload (Cubin)
and simply parse it as ELF data:

```rs
fn get_entry_elf<'a>(entry: &'a FatBinEntry, data: &'a [u8]) -> Result<ElfFile64::<'a, Endianness>, object::Error> {
    let FatBinEntry{ fatbin_info, payload_offset } = entry;
    let payload = &data[*payload_offset..fatbin_info.padded_payload_size as usize + *payload_offset];
    ElfFile64::<Endianness>::parse(payload)
}
```

Then we can easily list sections it contains with:

```rs
for (i, section) in elf.sections().enumerate() {
    res.push(String::from(section.name().unwrap_or("?")))
}
```

#### Example

By utilizing `fatbinspect` we can list all the sections in a Cubin. We can
already identify the sections that are particularly interesting for our
purposes: `.nv.constant0._Z4testPm`, `.text._Z4testPm` and `.nv.global.init`.

<script src="https://asciinema.org/a/Q9Efvhdfr8nBVOEB.js" id="asciicast-Q9Efvhdfr8nBVOEB" async></script>

### Modifying the binary

Finding the bytes is only half of the problem. In my earlier experiments, I
could identify an instruction and modify the corresponding bytes directly, but
the process was deliberately low-level: offsets had to be tracked manually and
there was no structural knowledge of the binary surrounding them.

Now that the Cubin is parsed as an ELF, `fatbinspect` knows which section owns
each range of bytes.

This gives the modification step an additional layer of safety. Instead of
treating the Cubin as an unstructured byte array, the tool can verify that an
address being modified belongs to the section we intended to operate on.

The following example applies the same kind of SASS modification demonstrated
in my previous NVIDIA reverse-engineering articles, but this time through the
new tooling:

<script src="https://asciinema.org/a/qZ7zCCnciDDDQwye.js" id="asciicast-qZ7zCCnciDDDQwye" async></script>

The resulting binary can then be used in the same way as the original, allowing
us to verify that the modification made it through the entire extraction and
reconstruction process.

## Final thoughts

`fatbinspect` gives me a much better foundation for the experiments that follow.
Instead of treating a CUDA binary as a collection of offsets and byte sequences,
I can now navigate it through the structures that actually contain those bytes.

The tool is still small and incomplete, but it turns the previous messy approach
into something much more precise and easier to build on. The next step is to use
that foundation for more experiments with NVIDIA SASS at the machine-code level.

The `fatbinspect` tool repository can be found on
[GitHub](https://github.com/StjepanPoljak/fatbinspect).

Research by Stealthium on Fatbin and Cubin binaries can be found
[here](https://stealthium.io/blog/fatbins-cuda-gpu-binary-formats-part-1).
