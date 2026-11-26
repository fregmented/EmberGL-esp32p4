# Phase 0 — 04: Candidate Optimizations and Benchmark Methodology

Covers **deliverable 6** (candidate optimizations) and **deliverable 8**
(benchmark methodology). **Phase 0 contains zero measured P4 performance
numbers** — no PPA/DMA2D throughput, no PSRAM bandwidth, no tile rate, no
concurrency scaling figure exists in this document or any Phase 0 document.
Everything below defines *what will be measured and how*, not results.

## 1. Candidate optimization inventory

Candidates are ranked by expected relevance to a tile-based software
rasterizer on a scalar RV32 core; each carries its evidence status. Nothing
here is implemented or measured.

| # | Candidate | Rationale | Status |
| --- | --- | --- | --- |
| O1 | Correct cache maintenance around tile writeback (`esp_cache_msync` C2M/M2C, right-sized rows) | Correctness first: DMA-read surfaces are unusable without it; the writeback window size directly affects present cost | Design-verified (02 rows 16–17); effect UNVERIFIED `[U]` |
| O2 | Render tiles directly into the driver-owned DPI framebuffer (in-FB draw fast path) | Avoids one full-tile copy per present; the driver msyncs only touched rows (`esp_lcd_panel_dpi.c:640-668`) | API-verified; behavior UNVERIFIED `[U]` |
| O3 | PPA for present-stage ops (fill/blend/scale-rotate) where formats align | Offloads scalar pixel loops; limited format/dimension envelope (ADR-0004) | API-verified; benefit UNVERIFIED `[U]` |
| O4 | DMA2D 2-D color-convert as tile transfer engine (`esp_async_color_convert`) | Strided async transfer with ISR completion; frees CPU during blit | API-verified; benefit UNVERIFIED `[U]` |
| O5 | Single-precision math hygiene in shaders (float throughout, avoid double promotion) | Hard float F/D exists; accidental `double` promotion halves throughput | Source fact (no SIMD, hard float); effect UNVERIFIED `[U]` |
| O6 | `-O2`/`-O3` and LTO comparisons for the rasterizer translation units | Template-generated rasterizers are large inlined code; codegen quality matters | NOT RUN |
| O7 | Instruction-cache friendliness: keep hot rasterizer loops out of flash thrash (IRAM placement is a candidate, not a decision) | 768 KB SRAM budget vs large template code | NOT RUN |
| O8 | Morton tile order interplay with PSRAM burst behavior | Upstream default is morton (`src/egl_rasterizer.h:88`); sequential vs morton writeback cost unknown on P4 | NOT RUN |
| O9 | Worker parallelism (2 cores, disjoint tiles) | Phase 5 scope; only after single-core determinism | NOT RUN |
| O10 | `esp_dsp` / PIE-assisted kernels | `esp_dsp` absent from IDF tree; PIE is asm-only — likely out of scope | NOT RUN, feasibility `[U]` |

Explicitly **not** candidates in early phases: any change to `src/`
(frozen, ADR-0001), assembly rewrite of the rasterizer, algorithm
replacement.

## 2. Deterministic scene suite

A fixed set of scenes exercising distinct code paths, so every number is
attributable:

- S1 solid clear/fill (pure present path),
- S2 opaque triangles, depthless (pure rasterizer),
- S3 depth-tested opaque, front-to-back sorted (Hi-Z stress),
- S4 alpha-blended, back-to-front (blend path),
- S5 textured quads, point filter,
- S6 textured quads, bilinear + clamp,
- S7 multi-dispatch mixed PSOs (dispatch/PSO-store churn),
- S8 worst-case overdraw cluster (Hi-Z reject-rate stress).

Rules: fixed seeds; fixed geometry converted to `.p3g`/`.ptx` once and
committed as fixtures (no runtime generation); identical camera and state
across runs; the suite runs identically on host and target.

## 3. Host golden reference

- Build the same scene suite on the host (x86-64 GCC/Clang and/or MSVC)
  against the unmodified upstream `src/`.
- The host output image per scene is the **golden reference**. Target
  correctness = pixel-compare against the golden reference.
- Rationale: the frozen `src/` is the single source of truth for expected
  pixels; port code may change where pixels come from, never what they are.

## 4. Tolerance policy

- **Integer render paths** (all integer pixel formats, depth `uint8/uint16`,
  Hi-Z): bit-exact match required. Rasterization is specified integer
  arithmetic (top-left rule, 4-bit sub-pixel precision); any difference is a
  bug, not tolerance.
- **Float-exposed paths** (float32 depth, float color formats): differences
  allowed only from declared floating-point reassociation, with a
  **per-channel declared tolerance** stated in the comparing tool's
  configuration before the run. Undeclared tolerance = failure. The
  declaration names the operation (e.g. barycentric interpolation order)
  that causes it.
- PPA/DMA2D-assisted paths must match their software twin under the same
  policy; format conversions the engines perform have their own declared
  tolerance (engine rounding is a hardware fact to be documented, not
  assumed).

## 5. Measurement procedure

For every timing measurement, all of the following are mandatory:

1. **Warmup**: run the scene N_warm times before timing (N_warm fixed per
   harness config, not guessed at report time).
2. **Fixed N**: a pre-registered sample count N; no early stopping.
3. **Statistics**: report median and p95 over the N samples (mean only as
   auxiliary); never a single best run.
4. **Raw samples**: keep the raw per-frame sample list in the measurement
   artifact; reports link it.
5. **Environment metadata** recorded with every result: IDF version, GCC
   version, target, board `[U]` until verified, silicon revision `[U]`,
   `sdkconfig` diff vs defaults (freq, SPIRAM, L2 size, compiler opts),
   clock config, power settings, frame size/format.
6. **Units and scope**: name exactly what is timed (e.g. rasterize-only vs
   rasterize+present) — mixed-scope numbers are invalid.
7. **Reproducibility**: harness and scene fixtures are versioned in the
   repository; a number without a rerunnable path is not publishable.

Every number is labeled MEASURED (this procedure, artifact linked),
ESTIMATED (arithmetic from verified facts), or UNVERIFIED (`[U]`). Absent
labels = not publishable.

## 6. Comparison matrix (to be filled by future measurements)

The standing structure for results; cells stay empty until real runs exist:

| Scenario | Scope timed | Single-core median | Single-core p95 | Dual-worker median | Engine-assisted median | Golden-match |
| --- | --- | --- | --- | --- | --- | --- |
| S1–S8 | rasterize-only | *(no data)* | *(no data)* | *(no data — Phase 5)* | *(no data)* | *(no data)* |
| S1–S8 | rasterize+present | *(no data)* | *(no data)* | *(no data)* | *(no data)* | *(no data)* |

No benchmark value tables exist in Phase 0 by design. The first filled
matrix must be accompanied by its raw-sample artifacts and environment
metadata per §5.
