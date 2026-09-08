# Changelog

All notable changes to `rusty_h265` are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project uses
[semantic versioning](https://semver.org/spec/v2.0.0.html).

## [0.6.0] - 2026-09-07

Conformance unchanged: **147/147** JCT-VC `HEVC_v1` bit-exact, SEI 100/100, on
both the AVX2 and SSE4.1 rungs. A memory-copy release: **1.078x** mainstream
inter, **1.086x** 10-bit, **1.158x** deblocking-heavy, **1.075x** weighted
prediction and **1.024x** all-intra over 0.5.0, measured against the 0.5.0 binary
in the same session with 21 pairs each -- every row a verdict. Weighted
prediction is now **1.05x FASTER than ffmpeg** (21/21, z = 4.58), the first
stream we win outright.

### Added

- **`tools/hevc/memcpy_census.py`** -- every `memcpy`/`memset`/`memmove`/
  `__rust_alloc` call the shipping build actually emits, attributed to its symbol
  and source line, with the length resolved when it is an immediate. Source grep
  finds only the copies you wrote; this finds the ones the optimiser made, and
  the ones whose RUNTIME length turned them into calls. It runs a self-test on
  every invocation, because a probe whose good news is a zero is indistinguishable
  from a broken one -- which happened: a `` collapsed into a literal backspace
  byte and the census cheerfully reported zero copies across four crates.

### Changed

- **The picture pool no longer clears the buffer it recycles.** At a 99% hit rate
  the per-picture stage still cost 3.4% of decode, all of it the 2.76 MB `memset`.
  Coverage is now tracked per coding tree block (set AFTER the block decodes, so a
  slice that dies mid-block leaves it uncovered) and only uncovered blocks are
  zeroed, before the in-loop filters read across block boundaries. Byte-identical
  on every path, not just conformant ones.
- **Three short-runtime-length copies dispatched to constant widths** -- the
  per-4x4 map writer (rows of 2 to 16 entries, 8-10 maps per coding unit, nine
  memsets inlined into `coding_quadtree`), the intra reference gather (a
  four-sample availability run at a time, now coalesced into spans), and the
  full-pel motion copy (rows of 8-128 bytes, 71,413 blocks a clip).
- **The intra reference filter runs in place.** §8.4.4.2.3 "cannot filter in
  place" only holds if you discard the one sample the 3-tap destroys; keeping it
  in a register retires two copies per filtered block, a whole write pass, and two
  128-byte buffers.
- **Four per-4x4 maps are no longer pre-filled** (230,400 bytes per picture of
  pure overwrite). Proven by poisoning each with a value that would change the
  output if read -- 147/147 -- against a control, `nz`, which is genuinely not
  covered and gives 19/147.
- **`ScalingFactors` is one flat block** instead of `Vec<Vec<Vec<u8>>>`, which
  allocated 29 times per picture for 8,160 bytes of sequence-invariant table.
- **The DPB purge is in place**, replacing a `drain(..).partition(..)` that
  allocated two vectors and moved every entry, twice per picture, to remove on
  average less than one.
- **Output serialisation narrows a row at a time.** It was converting 1,382,400
  samples per frame one by one; appending through a `Vec` puts a capacity check on
  every sample and blocks the vectorisation. Worth 5.4% of whole decode on a path
  no decoder benchmark covers, because they all decode to `-`.

### Fixed

- **The benchmark harness could be aborted by its own allocator guard.** Probing a
  foreign reference arm (ffmpeg) invokes it with our CLI's argument shape; on
  Windows PowerShell `2>&1` against a native command wraps stderr in error records,
  which under `Stop` killed the run before a single measurement. The guard's teeth
  are unchanged -- any arm that prints `alloc=` must print `rusty`.

## [0.5.0] - 2026-09-07

Conformance unchanged: 147/147 JCT-VC `HEVC_v1` bit-exact, SEI 100/100, on both
the AVX2 and SSE4.1 rungs. No measurable performance change -- this release is
about being able to trust the measurements.

### Fixed

- **The command line ignored flags that came before the paths.** The two
  positional arguments were read as `argv[1]` and `argv[2]` outright, so
  `rusty_h265 --isa sse41 in.bit out.yuv` tried to open `--isa` as a bitstream.
  Flags are now skipped wherever they appear. This is how it surfaced: a
  conformance run invoked as `--decoder "rusty_h265 --isa sse41"` reported
  **0/147**, which looks exactly like a decoder that has broken, on a change
  that touched no decoding.
- **`RH265_ISA=scalar` did not select the scalar kernels.** Level 0 is the SSE2
  rung and `scalar` was accepted as a synonym for `baseline`, so every "scalar"
  arm ever run executed vector kernels while announcing that it did not.
  Measured that way the whole vector path looked worth **3%**; with the `simd`
  feature genuinely off it is worth **2.81x** (7,554 ms against 2,685 ms on a
  20-second 720p clip). The variable now rejects `scalar` with a message
  pointing at `--no-default-features`. An arm that silently does not do what its
  name says is worse than no arm: every conclusion drawn from it is wrong, and
  confidently so.

### Added

- **`--isa avx2|sse41|baseline`** on the CLI, and `rusty_h265_accel::force_isa`
  behind it. `RH265_ISA` cannot do this job: both arms of a paired A/B run in
  the same environment, so setting it turns the comparison into a null arm
  without saying so -- the first attempt at measuring AVX2 against SSE4.1 did
  exactly that and read 1.017x, z = 0.30. With the flag: **AVX2 is 1.045x over
  SSE4.1** (14/15 z = 3.36, 15/15 z = 3.87).

### Notes

Two findings from this round, recorded here because they change what is worth
doing next rather than what the code does:

- **The kernels are worth 2.81x, and they are not width-limited.** Scalar ->
  SSE4.1 is ~2.7x; SSE4.1 -> AVX2 is 1.045x. The earlier reading that AVX2 and
  SSE4.1 are nearly equal was correct; the conclusion drawn from it -- "the
  vector loops are not where the time is" -- was not. The correct reading is
  that width above 128 bits buys almost nothing.
- **`unsafe` would not help.** The decoder core is `#![forbid(unsafe_code)]`, so
  the question is fair, and both probes are null: `panic = "abort"` measures
  0.990x/1.006x, and converting the hottest per-4x4 map reads to `get_unchecked`
  measures 0.994x/0.992x at 10/21, z = -0.22. The per-sample loops are already
  unsafe inside `rusty_h265-accel`; what the guarantee still covers is per-block
  glue entered ~1.2 M times a clip against the kernels' ~700 M samples. The
  safety boundary is already drawn where the cost is not.

## [0.4.0] - 2026-09-07

Conformance unchanged: 147/147 JCT-VC `HEVC_v1` bit-exact, SEI 100/100, on both
the AVX2 and SSE4.1 rungs.

### Added

- **A stage profiler** (`prof` feature, `RH265_PROF=1`). Roughly ninety wins had
  been landed using static instruction and guard-branch counts from the emitted
  assembly. Those count *work removed*; they are not a map of *where the time
  is*, and nothing had looked since the kernels landed. The report prints each
  stage's call count and the profiler's own measured per-scope tax beside its
  time, and flags any stage whose tax dominates it, because a profiler at
  millions of calls is part of the system under test.
- **An `x86-64-v3` build** (`tools/build-v3.{sh,ps1}`), measured 1.066x and
  bit-exact, CI-gated. See 0.3.0's notes and the README.

### Performance

Measured against the 0.4.0 binary's 0.3.0 predecessor, both under the shipping
allocator, both serialising, 15 pairs, ABBA-interleaved:

| stream | 0.3.0 | 0.4.0 | ratio | verdict |
|---|---:|---:|---:|---|
| x265-encoded 20 s 720p30 | 3,554 ms | 3,136 ms | 1.155x | 15/15, z = 3.87 |
| deblocking-heavy | 972 ms | 777 ms | 1.307x | 15/15, z = 3.87 |
| JCT-VC conformance | 392 ms | 344 ms | 1.111x | 15/15, z = 3.87 |

The profiler found both wins on its first run:

- **The sequence-invariant tables were rebuilt for every picture.**
  `MinTbAddrZs` and the raster-order tile map are functions of the SPS and the
  tile layout alone, so they are identical for every picture of a coded video
  sequence -- and building `MinTbAddrZs` walks every 4x4 block in the frame,
  57,600 of them at 720p, 600 times over for a 600-picture clip. They are now
  built once and shared. Per-picture setup fell from **13.7% of decode to 4.7%**.
- **The CLI serialised every frame into a buffer it then discarded.** With `-`
  as the output, `drain` still called `write_yuv` -- 1.38 MB of memcpy per
  frame, 830 MB over a 600-frame clip -- and wrote it to `io::sink()`. That is
  not decoding, `ffmpeg -f null -` does not do it either, and it was on the path
  every published timing measures. The untimed residue fell from **8.5% to 1.8%**.

Where the time goes now, on the 20-second clip: motion compensation 32%, entropy
and syntax 17%, inverse transform 14%, deblocking 12%, SAO 9%, per-picture setup
5%. The next structural item is the intermediate `pred` buffer in MC -- the
interpolation writes `i16`, then `put_uni`/`put_bi` reads it back to write the
picture, a whole pass that could be fused into the vertical filter.

### Fixed -- the harness, again

- **`-RefMsScale` had been reverted by a bad copy-back**, so the harness shipped
  in 0.3.0 could not reproduce the ffmpeg table in its own README (ffmpeg's
  `-benchmark` prints `rtime=0.198s`, which needs scaling to ms).
- **The allocator guard checked only one arm.** Our binaries run under
  `rusty_alloc` in production and a system-allocator build is not comparable;
  the harness enforced that for `-Ours` and never for `-Reference`. A release
  gate's `cargo test --release` had relinked the 0.3.0 reference without
  `bench-alloc`, and the resulting mismatch manufactured **1.61x-2.20x at 15/15,
  z = 3.87** -- verdict-strength, and entirely an artefact. It was caught only
  because the profiler predicted 1.13x and the arithmetic did not agree. Both
  arms are now checked. This is the second time this exact trap has fired.

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
