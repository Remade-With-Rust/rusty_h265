# Changelog

All notable changes to `rusty_h265` are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project uses
[semantic versioning](https://semver.org/spec/v2.0.0.html).

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
