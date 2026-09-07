# Changelog

All notable changes to `rusty_h265` are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project uses
[semantic versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - 2026-09-07

A performance release, and a **correction to how we measure**. Conformance is
unchanged and still exact: 147/147 JCT-VC `HEVC_v1` streams bit-exact, SEI
100/100, on both the AVX2 and SSE4.1 rungs.

### Fixed -- the benchmark harness, and the numbers it produced

- **`tools/bench/codec-bench.ps1` timed with `Process.TotalProcessorTime`, which
  on Windows is kernel TICK ACCOUNTING quantised to 15.625 ms.** Every reading in
  the 0.2.0 performance table was an exact multiple of that tick, so the
  differences it reported were two ticks; anything smaller was invisible, and the
  harness's tie-exclusion then discarded exactly the closest pairs. On one stream
  the paired-ratio and per-arm-median estimators disagreed in SIGN because both
  were reading a 15.6 ms lattice. Timing is now the arms' own internal decode time
  (1 ms resolution) or a `QueryPerformanceCounter` wall clock, and the harness
  measures its own quantum and refuses to report a difference under three of them.
- **It charged process startup to each arm.** `ffmpeg.exe` is 242 MB against our
  696 KB, so ffmpeg was paying ~170 ms of image loading on a ~280 ms decode. Both
  arms now report decode time from inside the process.
- **CPU time is still collected, for the job it is actually good at**: `cpu/wall`
  per sample, which detects a descheduled (< 1) or multi-threaded (> 1) run. That
  was the real content of the rule that made it the timing quantity.
- **Consequently the published ffmpeg ratio moves from 1.3x-1.8x to 1.4x-2.0x.**
  That is a measurement fix, not a regression -- the decoder got faster this
  release, on every stream where the result is a verdict. The corrected method is
  less flattering to us, which is why it is the one we publish.

### Performance

Measured directly against the 0.2.0 binary on the same box, corrected harness,
21 pairs, ABBA-interleaved:

| stream | 0.2.0 | 0.3.0 | ratio | verdict |
|---|---:|---:|---:|---|
| 720p 8-bit, mainstream inter | 574 ms | 541 ms | 1.059x | 19/21, z = 3.71 |
| deblocking-heavy | 1,428 ms | 1,380 ms | 1.031x | 20/21, z = 4.15 |
| weighted prediction | 1,084 ms | 978 ms | 1.034x | 20/21, z = 4.15 |
| all-intra | 6,848 ms | 6,777 ms | 1.018x | 14/21, z = 1.53 -- not a verdict |

- **The motion-compensation entry path**: `interp_luma` went 500 -> 346 emitted
  instructions and `interp_chroma` 423 -> 254, with the vector kernels untouched.
  Fixed-size filter tables with masked indices retire the bounds checks (guard
  branches 20 -> 5 and 13 -> 2); one dispatcher read per call is threaded down
  instead of three `OnceLock` probes; the never-taken scalar fallbacks moved off
  the hot path.
- **The loop filter**: `apply_in_loop_filters` went 2,943 -> 2,420 instructions.
  `filter_chroma_edge` takes the segment's whole footprint as one window per
  direction rather than re-proving twelve indexed accesses; the boundary-strength
  scan reads a row slice of each per-4x4 map instead of indexing five `Vec`s per
  block; the q block's CTB and filter parameters are derived once for both
  directions.
- **The residual parser and scan tables**: five masked indices, and two more
  per-block table dispatches folded into the single `ScanSet` lookup.
- **Six pixel-kernel scalar fallbacks** marked cold as one set -- `weighted_uni`
  205 -> 112 instructions, `put_bi_fp` 297 -> 226, `avg_block` 209 -> 149.

### Rejected

Thirteen candidate optimisations were built, measured and reverted, and the
reasoning is in `docs/plans/` upstream. Three worth naming: fusing `slice_addr`
and `tile_id` into one per-CTB key cost +164 instructions as an extra array and
+259 as a replacement (the fusion itself is what does not pay); row-slicing a
per-block map won on a dense scan and LOST on a strided one in the same function;
and marking `copy_block_scalar` cold took its caller to one instruction, which is
a bare jump rather than a win -- `copy_block` has no vector path, so the
"fallback" is its only path.

## [0.2.0] - 2026-09-07

A performance release. Conformance is unchanged and still exact; the decoder is
roughly a third of the way closer to ffmpeg than 0.1.0 was.

### Performance

- **1.3x-1.8x of ffmpeg 8.1.2**, from 1.5x-2.3x in 0.1.0, on the same harness
  and the same box. The 720p 8-bit mainstream row went 781 ms -> 563 ms.
- **The entropy coder and the syntax layer around it**: 53 measured,
  individually gated changes. The largest were structural rather than clever --
  the residual parser cleared a whole transform block when the
  last-significant-coefficient position said a fraction of it was live (178.8 M
  `i32` stores down to 67.3 M on intra-heavy content); two linear searches of
  the scan tables became lookups; five per-block scan-table dispatches became
  one; §6.4.1 availability was split so a block's own half is computed once
  instead of once per neighbour; and `wrap_qp` stopped issuing a hardware
  signed division on every coding unit.
- **The inverse transform** gained a SIMD kernel and `i32` accumulators (the
  `i64` ones were never needed -- the bound is 61,014,016 against `i32`'s 2.1 G).
- **Deblocking** gained a kernel; the stage halved from 13.3% of decode to 6.5%.
- Six candidate optimisations were measured and **rejected**, four of them the
  same shape: masking away a bounds check trades a never-taken branch for a
  bigger loop body.

### Changed

- `Contexts` now holds `[u8; CTX_PAD]` (256) rather than `[u8; NUM_CTX]` (157).
  The padding makes `decode`'s index guard a single mask instead of a compare
  and a conditional move, on a path that runs about four million times per 720p
  frame-set. **This is the breaking change that makes this 0.2.0.**
- `PicState` gains `avail_at` / `avail_n` / `avail_n_idx`, the split form of
  `available`; `available` itself is unchanged and still present.
- New `cabac-trace` feature (default off) carries the per-bin CABAC trace that
  used to be a runtime flag, so no shipping decode pays for the branch.

### Added

- `rusty_h265-accel` gains `itx`, the inverse-transform kernel, which 0.1.0
  shipped without.

## [0.1.0] - 2026-09-06

First public release: a complete, conformance-verified HEVC decoder.

### Conformance

- **147/147** JCT-VC `HEVC_v1` streams decode byte-for-byte identical to the
  reference, compared by MD5.
- **6,183/6,183** decoded-picture-hash SEI messages verified, so every picture
  carrying the encoder's own MD5 is checked against it.
- `HEVC_REQUIRE_VECTORS=1` turns the "corpus absent, skip" path into a hard
  failure, so the gate cannot pass having verified nothing. A `HEVC_ONLY` typo
  once selected zero streams and went green in 0.01 s; the guard exists because
  of that.

### Decoder

Main, Main 10 and Main Still Picture, 8- and 10-bit, 4:2:0. Full CTU quadtree,
all 35 intra modes, merge/AMVP inter prediction, weighted prediction, DCT/DST
inverse transforms with transform-skip, deblocking, SAO, PCM, tiles, wavefronts,
long-term references and the RPS/DPB machinery. CABAC with the LPS-range and
state-transition tables fused into one branchless step.

Range Extensions, SHVC, 3D-HEVC and screen content are **rejected at the SPS**
rather than silently mis-decoded.

### Acceleration

`rusty_h265-accel` holds every kernel and is the only crate permitted `unsafe`;
the decoder core is `#![forbid(unsafe_code)]`. Kernels cover motion compensation
(FIR, full-pel copy, bi-prediction, weighted prediction), intra (angular,
planar, DC, transposed), the inverse transform, SAO (band and edge) and
deblocking. Each has a scalar twin kept permanently as its differential-test
oracle and its fallback.

Notable optimisations in this release:

- **Partial butterfly** for the inverse DCT: 1,024 multiplies become 352 at
  32-point, by exploiting the matrix's even/odd symmetry.
- **DC-only collapse**: a block with one non-zero coefficient degenerates to a
  scalar and a fill, because row 0 of the DCT matrix is constant.
- **Deblocking vectorised** end to end -- the stage went from 13.3% of decode to
  6.5%.
- **Symmetric-filter fold** in the vertical MC kernel, and a sign flip in the
  angular kernel that lets a load fold into an operand.

### Correctness guards

- Every SIMD kernel is gated on **bit-identity** with its scalar twin, not a
  tolerance.
- A **census** (`RH265_CENSUS=1`) counts which arm each call takes, with a test
  asserting every declared counter is actually incremented -- a counter reading
  zero forever is indistinguishable from a cold path, and that ambiguity once
  caused a real kernel to be pruned as unreachable while it served 100% of
  motion compensation on one stream.
- The `i16` kernels that are sound only because the SPS rejects bit depths above
  10 now **enforce that bound themselves** and fall back to the scalar twin,
  so enabling Range Extensions would cost speed rather than correctness.

[0.1.0]: https://github.com/Remade-With-Rust/rusty_h265/releases/tag/v0.1.0
