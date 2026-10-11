# Content-Aware Scale and region I/O (2026-10-11)

Scenario P28 in [`docs/scorecard.md`](scorecard.md#performance): Edit › Content-Aware Scale of a
24 MP (6000 × 4000) photo to 90 % width, budget 3000 ms ([#211](https://github.com/storytold/photocraft/issues/211)).
The committed baseline (2026-10-05, Apple M4 Pro under a load average of ~225) reads 954 s; the
tall-image seam-search proxy already in `crates/algo/src/seam.rs` brought it to ~4 s before this
change.

Machine for every number below: 4-core x86-64 cloud VM, no GPU (`--cpu`), release build,
`perf_scenarios --cpu --group algorithms --only Content-Aware`. These are not baseline numbers
(different machine class); `cargo xtask perf --update-baseline` on the reference Mac remains the
way to move the scorecard.

## Where the time went (before)

| Step | ms |
|---|---:|
| Read the layer region as floats (`Surface::read_region`) | 282 |
| Seam carving (`carve_width_proxy_height`) | ~2170 |
| of which: seam search (dynamic programming, full map per seam) | 815 |
| of which: shifting rows after each removed seam | 381 |
| of which: applying the seams to the full-height image (serial) | ~300 |
| Clear the old area (`pixels::try_clear_surface`: read, modify, write) | 1170 |
| Write the result (`Surface::write_region`) | 168 |
| **Total (p50)** | **3992** |

## Changes

1. **Clearing without a selection fills instead of read-modify-write**
   (`crates/engine/src/pixels.rs`). With no selection every pixel becomes the empty pixel, so
   `try_clear_surface` now calls the new fallible `Surface::try_fill_rect` (whole tiles share one
   solid tile, which `prune` drops). 1170 ms → 71 ms. Also used by Cut and by Warp's
   "rest of the layer". Test: `clearing_without_a_selection_matches_the_masked_clear` (pixels,
   bounds and tile count equal the masked path; the undo snapshot is untouched).
2. **Incremental seam search** (`Carver::find_seam`). The cumulative cost and parent maps are
   kept between seams and shifted with the image; after a seam is removed only the cone it can
   reach is recomputed (the energy span, a window around the seam where a pixel's parents
   straddle it, and the columns whose cost changed in the row above). Results are bit-identical
   to the full search: `incremental_seam_search_matches_full_search` compares every seam of 60
   random wide, tie-heavy images against `Dp::seam`, and the existing
   `carver_matches_reference` oracle still passes. On the 24 MP case the search visits 33 M cells
   instead of ~1.75 G.
3. **Parallel proxy build and seam application** (rayon, serial on wasm). The apply step copies
   the runs between removed columns instead of pushing pixel by pixel. Output hash on a
   6000 × 4000 × 3 test image is unchanged (`d9701258d583ef1c` before and after).
4. **Row-at-a-time sample conversion** (`photocraft_color::read_samples` / `write_samples`, used
   by `read_region`, `read_region_into`, `write_region` and `try_write_region`). The sample type
   is matched once per row run instead of per sample. Bit-identical
   (`sample_runs_match_single_samples`); 15–45 % faster per call on 24 MP RGBA at 8/16/32 bits.
   These calls are memory-bound on this VM, so the gain is modest; they sit under most filters
   and edits.

## Result

| | Before | After |
|---|---:|---:|
| Content-Aware Scale 24 MP → 90 % (p50, 3 runs) | 3992 ms | **1966–1980 ms** |
| `carve_width` alone, 6000 × 4000 × 3 | 1822 ms | 948 ms |

## Still open

- Row shifting after each seam is now the largest carving cost (~460 ms: about 25 GB of
  `copy_within` over 600 seams). Shifting the shorter side of each row (a per-row offset)
  would roughly halve it.
- `read_region` / `write_region` are single-threaded and dominated by page faults on fresh
  buffers; a parallel tile path would help every caller.
- Select › Modify › Feather (r 50) measures 120–130 ms here against 100 ms, Smooth 95–107 ms:
  the blurs are already radius-independent and parallel (55–90 ms); the rest is mask read
  and selection write (~40 ms).
