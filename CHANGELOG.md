# Changelog

All notable changes to `rusty_h265` are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project uses
[semantic versioning](https://semver.org/spec/v2.0.0.html).

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
