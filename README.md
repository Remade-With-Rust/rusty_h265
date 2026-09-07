# rusty_h265

[![crates.io](https://img.shields.io/crates/v/rusty_h265?logo=rust)](https://crates.io/crates/rusty_h265)
[![docs.rs](https://img.shields.io/docsrs/rusty_h265?logo=docsdotrs)](https://docs.rs/rusty_h265)
[![CI](https://github.com/remade-with-rust/rusty_h265/actions/workflows/ci.yml/badge.svg)](https://github.com/remade-with-rust/rusty_h265/actions/workflows/ci.yml)
[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)
[![Remade With Rust](https://img.shields.io/badge/Remade%20With-Rust-000?logo=rust&logoColor=fff)](https://github.com/remade-with-rust)
[![By Mata Network](https://img.shields.io/badge/by-Mata%20Network-5b2be0)](https://www.mata.network)

> **rusty_h265** is a ground-up, pure-**Rust** HEVC / H.265 **decoder**: a
> `#![forbid(unsafe_code)]` decoder core, permissively licensed, with no C and no
> FFI. It was written from the ITU-T H.265 specification, not ported from
> libde265 or ffmpeg, and it decodes **all 147 streams of the JCT-VC `HEVC_v1`
> conformance suite byte-for-byte identical** to the reference.

Part of **[Remade With Rust](https://github.com/Remade-With-Rust)** by
**[Mata Network](https://www.mata.network/)** -- the HEVC decoder inside
**[remade_ffmpeg_rs](https://github.com/Remade-With-Rust/remade_ffmpeg_rs)**, our
memory-safe FFmpeg alternative, alongside
**[rusty_h264](https://github.com/Remade-With-Rust/rusty_h264)**.

## The headline

- **147/147 conformance streams bit-exact.** The full JCT-VC `HEVC_v1` suite,
  compared by MD5 against the reference decoder -- not "visually identical", not
  PSNR. Byte for byte.
- **6,183/6,183 decoded-picture-hash SEI messages verified.** Where a stream
  carries the encoder's own per-picture MD5, we check it -- the codec grading its
  homework against the encoder's, on every picture.
- **The decoder core is `#![forbid(unsafe_code)]`.** Every raw pointer and
  `target_feature` lives in one small crate, `rusty_h265-accel`, behind a
  runtime-dispatched seam. `--no-default-features` compiles a decoder with zero
  `unsafe` in the entire dependency tree.
- **Every kernel has a scalar twin**, kept forever, differential-tested for
  **bit-identity** rather than a tolerance. HEVC is exact; anything less is a bug.

| | libde265 / ffmpeg (C) | **rusty_h265 (Rust)** |
|---|---|---|
| C/C++ in the dependency tree | all of it | **none** |
| `unsafe` in the decoder core | extensive | **0** -- `#![forbid(unsafe_code)]` |
| Dependencies | several | **zero** (one optional, benchmark-only) |
| `HEVC_v1` conformance | -- | **147/147 byte-exact** |
| Picture-hash SEI verified | -- | **6,183/6,183** |

### Performance

Measured against **ffmpeg 8.1.2's native `hevc` decoder** -- hand-written
assembly, the fastest widely-available software HEVC decoder, and a deliberately
tougher bar than libde265.

| stream | Mpx | ffmpeg 8.1.2 | **rusty_h265** | ratio | paired verdict |
|---|---:|---:|---:|---:|---|
| 720p 8-bit, mainstream inter | 55.3 | 277 ms | 539 ms | 0.51x | 1/21, z = -4.15 |
| 720p 10-bit, same content | 55.3 | 321 ms | 652 ms | 0.51x | 0/21, z = -4.58 |
| deblocking-heavy | 331.8 | 738 ms | 1,313 ms | 0.56x | 0/21, z = -4.58 |
| weighted prediction | 255.6 | 766 ms | 1,097 ms | 0.70x | 0/21, z = -4.58 |

<sub>**We are 1.4x-2.0x slower than ffmpeg, and that is the honest number.**
Measured 2026-09-07 with [`tools/bench/codec-bench.ps1`](tools/bench/codec-bench.ps1):
pinned to one core at High priority, arms ABBA-alternated, 21 pairs, paired
win-rate with a z-score -- every row is a verdict (|z| > 2), not noise. Both arms
discard their output, and **both arms' decoded frame counts are checked against
`ffprobe` before any timing is reported**. The resolution floor -- ffmpeg measured
against itself -- is **0.980x**.</sub>

<sub>**This supersedes the 1.3x-1.8x published for 0.2.0, and the change is a
measurement fix, not a regression.** That figure came from two defects in our own
harness, both corrected in this release. (1) It timed with
`Process.TotalProcessorTime`, which on Windows is kernel TICK ACCOUNTING quantised
to **15.625 ms** -- every reading in the 0.2.0 table was an exact multiple of it,
so the differences quoted were two ticks, and the harness's tie-exclusion then
silently dropped the closest pairs. (2) It charged each arm's process startup to
that arm, and `ffmpeg.exe` is a **242 MB** binary against our 696 KB, so ffmpeg
paid ~170 ms of image loading on a ~280 ms decode. Both arms now report their own
internal decode time (`decode_ms` here, `-benchmark rtime` for ffmpeg), which has
1 ms resolution and excludes process launch on both sides. The corrected method is
LESS flattering to us, which is why it is the one we publish.</sub>

<sub>Release over release, measured directly against the previous binary on the
same box with both arms under the shipping allocator: **0.4.0** is **1.155x** on an
x265-encoded 20 s clip, **1.307x** deblocking-heavy and **1.111x** on conformance
(15/15, z = 3.87 on each) over 0.3.0; 0.3.0 was 1.059x / 1.031x / 1.034x over
0.2.0. This decoder started at 3,750 ms on the first row.</sub>

<sub>0.4.0's two wins both came from the first run of the new stage profiler
(`--features prof`, `RH265_PROF=1`), after roughly ninety wins had been landed on
static instruction counts alone: the sequence-invariant scan tables were being
rebuilt for every picture (a walk over all 57,600 4x4 blocks, 600 times), and the
CLI was serialising every frame into a buffer it then discarded. Per-picture setup
fell 13.7% -> 4.7% of decode; the untimed residue 8.5% -> 1.8%. The profile now
reads: motion compensation 32%, entropy and syntax 17%, inverse transform 14%,
deblocking 12%, SAO 9%.</sub>

### A faster build for modern CPUs

The SIMD kernels already run AVX2 wherever the CPU allows -- they carry
`#[target_feature]` and are selected at runtime. Everything **else**, which is
roughly 87% of decode time (entropy coding, syntax, per-block bookkeeping),
compiles for the portable x86-64 baseline, i.e. SSE2, because the binary has to
run anywhere.

Building the whole decoder for `x86-64-v3` lets that majority use VEX encoding,
three-operand forms and sixteen `ymm` registers as well:

```sh
RUSTFLAGS="-C target-cpu=x86-64-v3" cargo build --release
# or: tools/build-v3.sh   (toolsuild-v3.ps1 on Windows)
```

| stream | portable build | `x86-64-v3` | paired verdict |
|---|---:|---:|---|
| x265-encoded, 20 s 720p30 | 3,442 ms | **3,008 ms** | 1.066x, 12/15, z = 2.32 |
| JCT-VC conformance | 367 ms | **344 ms** | 1.066x, 14/15, z = 3.36 |

<sub>Same harness and method as the table above. Output is **bit-identical** to
the portable build, and CI gates that on every push. `x86-64-v3` requires AVX2 +
BMI1/2 + FMA -- Haswell (2013) and later, Excavator (2015) and later -- and will
`SIGILL` on anything older, so it is a second artifact rather than a change to
the default one. `-C target-cpu=native` is faster still on a newer machine, and
is not distributable.</sub>

## What is this?

A complete HEVC **decoder**, written from the specification:

- **Profiles:** Main, Main 10, Main Still Picture. 8-bit and 10-bit, 4:2:0.
- **Coding tools:** the full CTU quadtree, all 35 intra modes including the
  strong-smoothing filter, inter prediction with merge/AMVP, weighted
  prediction, the DCT/DST inverse transforms with transform-skip, in-loop
  deblocking and SAO, PCM, tiles, wavefronts, long-term references, and the
  RPS/DPB machinery.
- **Entropy:** CABAC, with the LPS-range and state-transition tables fused into
  a single branchless step.
- **Not supported**, and rejected explicitly at the SPS rather than silently
  mis-decoded: Range Extensions (12-bit, 4:2:2, 4:4:4), SHVC, 3D-HEVC, and
  screen-content coding.

## Install

```toml
[dependencies]
rusty_h265 = "0.1"
```

```bash
# SIMD kernels on by default (runtime-dispatched; no nasm, no FFI, no build script)
cargo add rusty_h265

# ...or 100% safe, portable Rust with zero `unsafe` compiled in:
cargo add rusty_h265 --no-default-features
```

## Quick start

```rust
use rusty_h265::Decoder;

let mut dec = Decoder::new();
for nal in annex_b_nal_units(&bitstream) {
    if let Some(pic) = dec.decode_nal(nal)? {
        // pic.planes[0] is luma, [1]/[2] chroma, each `&[u16]` with a stride.
        println!("{}x{} {}-bit", pic.width, pic.height, pic.bit_depth);
    }
}
```

The bundled CLI decodes a raw Annex-B stream to planar YUV, or to `-` to discard
(which is what the benchmark harness times):

```bash
cargo run --release --bin rusty_h265 -- input.hevc output.yuv
```

## Architecture

```text
  crates/rusty_h265           the decoder.  #![forbid(unsafe_code)]
        |
        |  runtime-dispatched seam: Scalar | Baseline (SSE2/NEON) | AVX2
        v
  crates/rusty_h265-accel     the kernels.  the ONE crate allowed `unsafe`
```

Every kernel in `-accel` is paired with a scalar twin that stays in the tree
forever -- as the differential-test oracle, and as the fallback on any CPU
without the instruction set. The seam carries a **census** (`RH265_CENSUS=1`)
counting which arm each call actually took, because a kernel with a test and a
benchmark but no production caller passes every gate while serving nothing.

## Verification

| gate | what it proves |
|---|---|
| `HEVC_v1` 147/147 | every conformance stream decodes byte-for-byte |
| picture-hash SEI 6,183/6,183 | per-picture MD5 matches the encoder's own |
| `*_matches_scalar` | each SIMD kernel is bit-identical to its scalar twin |
| census guards | every declared counter is actually incremented -- one that reads 0 forever is indistinguishable from a cold path |

```bash
# the full corpus (fetch the JCT-VC HEVC_v1 suite into hevc-vectors/ first)
HEVC_REQUIRE_VECTORS=1 cargo test --release
```

`HEVC_REQUIRE_VECTORS=1` turns the "no corpus, skip" path into a hard failure, so
CI cannot go green having verified nothing.

## Roadmap

- Range Extensions (12-bit, 4:2:2, 4:4:4) -- the profile check that rejects them
  today is the placeholder.
- Multi-threaded decode: tiles and wavefronts are parsed, but the pipeline is
  single-threaded.
- Closing the remaining gap to ffmpeg. MC, intra, deblocking and SAO are
  vectorised; the largest remaining scalar stage is CABAC, which is serial by
  construction.

## License

Apache-2.0. See [LICENSE](LICENSE).

## About Mata Network

[Mata Network](https://www.mata.network/) builds memory-safe, permissively
licensed media infrastructure in Rust. **Remade With Rust** is its open-source
home.
