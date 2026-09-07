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
| 720p 8-bit, mainstream inter | 55.3 | 328 ms | 563 ms | 0.60x | 0/15, z = -3.87 |
| 720p 10-bit, same content | 55.3 | 375 ms | 672 ms | 0.56x | 0/15, z = -3.87 |
| deblocking-heavy | 331.8 | 781 ms | 1,313 ms | 0.58x | 0/15, z = -3.87 |
| weighted prediction | 255.6 | 766 ms | 953 ms | 0.77x | 2/15, z = -2.84 |

<sub>**We are 1.3x-1.8x slower than ffmpeg, and that is the honest number.**
Measured 2026-09-07 with [`tools/bench/codec-bench.ps1`](tools/bench/codec-bench.ps1):
pinned to one core at High priority, **CPU time** (not elapsed), arms
ABBA-alternated, 15 pairs, paired win-rate with a z-score -- every row is a
verdict (|z| > 2), not noise. Both arms discard their output, and **both arms'
decoded frame counts are checked against `ffprobe` before any timing is
reported**. The resolution floor -- ffmpeg measured against itself -- is
**0.955x**. This decoder started at 3,750 ms on the first row; it is 563 ms now.
0.1.0 measured 0.44x-0.67x on the same harness, so the gap has closed by roughly
a third. (The deblocking-heavy row is not comparable across the two releases:
that stream was regenerated and is now 331.8 Mpx against 8.1 Mpx before. On the
current stream, 0.2.0 is 1.079x faster than 0.1.0, 20/21, z = 4.15.)</sub>

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
