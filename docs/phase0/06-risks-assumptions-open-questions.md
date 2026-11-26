# Phase 0 — 06: Risks, Assumptions, and Open Questions

Covers **deliverable 11**.

**Opening statement:** no ESP32-P4 performance, hardware, PSRAM, panel, or
accelerator number was measured in Phase 0. Nothing in this document is
backed by a hardware run; every hardware-dependent statement is marked
`[U]`. This document inventories what could go wrong, what we are taking on
faith, and who must answer what before which phase proceeds.

## 1. Risks (R1–R20)

Each risk lists trigger (how we would notice) and mitigation.

| ID | Risk | Trigger | Mitigation |
| --- | --- | --- | --- |
| R1 | **Baseline drift**: fork `develop` diverges unnoticed from upstream `v0.1.1` in protected paths | Traceability diff gate exits non-zero unexpectedly | Gate re-run on every PR (ADR-0001); never bypass; treat any hit as an incident |
| R2 | **`src/` edits** made under deadline pressure despite the freeze | Review sees diffs under `src/` | Ledger entry + owner approval is the only path; agents reject the task otherwise (AGENTS.md boundaries) |
| R3 | **Ledger staleness**: ledger entries no longer match actual deviations after upstream merges | Ledger audit finds entries without diffs or diffs without entries | Ledger review at every upstream re-baseline; ledger schema includes verification evidence |
| R4 | **Component proposal rejected/unworkable**: ADR-0002 cannot be implemented (naming, registry, target gating) | Phase 1 spike fails to register/build the root component | Fallback model: consume via example-local source list (no component); decide before Phase 1 code |
| R5 | **C ABI instability**: the `extern "C"` boundary keeps changing as Phase 1/2 learn | Repeated breaking changes to `port/c` headers | Freeze boundary after Phase 1 spike; changes require ADR amendments |
| R6 | **MISRA scope explosion**: required-rule deviations multiply, eroding the safety baseline | Deviation count grows per module review | Deviations must be local/documented/narrow (CONVENTION §4); recurring deviations force a design fix, not more deviations |
| R7 | **IDF API drift**: IDF 6.1.x patches rename/deprecate driver APIs (precedent: `esp_async_memcpy_new` gone in 6.1.1) | CI build failures on IDF minor bumps | Pin 6.1.1 as validated; treat IDF bumps as changes with their own verification; never assume API existence without header check |
| R8 | **ISA/SIMD expectation mismatch**: plans assume vector throughput the scalar RV32 core cannot deliver | Phase 4 measurements miss expectations by design margins | No vector claims anywhere (02 row 10); performance work only under the measurement discipline |
| R9 | **SRAM budget overrun**: statics + tile buffers + stacks exceed the ESTIMATED ~620 KB usable on rev ≥ v3 (768 KB − 128 KB L2 carve-out − ~20 KB reservation; rev < v3 gives ≈236 KiB) | Link-time region overflow at Phase 1 | ESTIMATED footprint accounting in `03` §4; recompute per config and per silicon revision; move nothing to PSRAM silently |
| R10 | **PSRAM absent/disabled**: board lacks PSRAM or `CONFIG_SPIRAM` stays off; DPI FB allocation fails (`ESP_ERR_NO_MEM`) | Panel creation fails at boot | Explicit config requirement documented; capability report exposes PSRAM presence; fail loudly, never silently shrink |
| R11 | **Cache coherency defects**: missing C2M/M2C produces intermittent corruption that "works on dev, fails in prod" | Golden-reference mismatches that move with timing | Normative cache rules (AGENTS.md); contract checks at the boundary; software-twin diffs |
| R12 | **Alignment hardcoding**: 64 B assumed; breaks when L2 line = 128 B (512 KB L2) or MSPI/DMA2D alignment applies | Crashes/corruption after config change | Derive allocator alignment from the build config (`max(L1, L2)` external, L1 internal-only), verify post-allocation with `esp_cache_get_line_size_by_addr()` (non-zero, divides address and size); grep ban on hardcoded alignment constants in `port/`; private helpers remain a Q10 decision |
| R13 | **PPA limit rejections**: dims > 16383, unsupported formats (A8 SRM), misaligned output size → runtime failures | Engine calls return errors in Phase 3 testing | Capability gate checks every constraint before submission (ADR-0004 §Decision 2); `force` fails explicitly |
| R14 | **DPI framebuffer ownership conflicts**: port writes into a driver-owned FB the driver reallocates or double-presents | Tearing/corruption when `num_fbs` > 1 | FB ownership contract in `03` §3.2; obtain FB via `esp_lcd_dpi_panel_get_frame_buffer()`; never cache FB pointers across reconfig `[U]` |
| R15 | **`EGL_DEBUG` shipped to device**: default debug build traps (`__builtin_trap()`) on any assert | Device panics during bring-up | Phase 1 must select release/retail config explicitly (`01` §6); no silent config inheritance |
| R16 | **Static tile-shader wrapper hazard**: second device instance of the same specialization races `wrapper.m_dev` (`src/egl_device.h:319-331`) | Two-instance test corrupts tile shader target | Single-instance rule until redesigned; Phase 5 must resolve before workers |
| R17 | **Vcache races under workers**: shared 32 KB vcache without locking | Bit-exact diffs only under dual-worker runs | Locking design required in Phase 5; determinism harness gates the feature |
| R18 | **Benchmark validity**: numbers measured on a config that differs silently from the documented environment | Results unreproducible by others | Measurement procedure §5 (`04`): fixed N, warmup, median/p95, raw artifacts, env metadata |
| R19 | **Board leakage**: panel/touch/board specifics creep into `port/` or docs | Review finds board names/pinouts in repo-owned layers | Scope rule (AGENTS.md); scaffold deps named only as scaffold deps; example stays the only board-touching place |
| R20 | **Patch-ledger bypass via tooling**: bulk tooling (formatters, codemods) rewrites `src/` en masse | Gate fails after an "innocent" tool run | Agents never run repo-wide formatters; gate in every PR; tools list restricted in AGENTS.md |

## 2. Assumptions (A1–A10)

Each assumption lists how it gets validated.

| ID | Assumption | Validation |
| --- | --- | --- |
| A1 | Upstream core behavior is deterministic per (input, config) | Host golden-reference runs: repeated runs byte-identical (`04` §3) |
| A2 | The frozen `src/` compiles unmodified for the P4 target | MEASURED in Phase 0: `-fsyntax-only` at gnu++11/17/20/26 (`01` §6); Phase 1 completes a real link |
| A3 | Tile buffers must stay in internal SRAM; PSRAM latency would dominate per-tile access | Phase 1/2 measurement: tile loop with buffers in each memory class |
| A4 | DPI in-FB draw (pointer inside driver FB) performs no copy, only row writeback | Hardware run verifying per-row msync behavior (`02` row 19 `[U]`) |
| A5 | `CONFIG_SPIRAM=y` is sufficient for DPI panel creation on the target board | Hardware run with a real panel `[U]` |
| A6 | Two FreeRTOS tasks pinned to the two cores can saturate a rasterization pipeline | Phase 5 benchmark, single- vs dual-worker matrix |
| A7 | EmberGL format→fourcc mapping (r5g6b5/b5g6r5 ↔ RGB16, b8g8r8a8 ↔ BGRA32) is a pure byte-order question | Hardware color-swatch test before any panel claim |
| A8 | The example's managed components (ST7701 panel, GT911 touch) are scaffolding, not supported hardware | Scope rule; no doc claims support; board bring-up is application work |
| A9 | IDF 6.1.1 driver behavior (msync directions, PPA alignment checks) matches the headers as read | Phase 3 engine tests assert the documented contract |
| A10 | Component-registry publication of an `embergl` component is possible at Phase 7 | Registry name-availability spike before Phase 7 `[U]` |

## 3. Open questions (Q1–Q12)

Each question names who can answer and which phase it blocks.

| ID | Question | Who can answer | Blocks |
| --- | --- | --- | --- |
| Q1 | Which build configuration ships by default (`EGL_DEBUG` vs `EGL_RELEASE`/`EGL_RETAIL`) for the example and the component? | Owner | Phase 1 |
| Q2 | Is ADR-0002 (root as IDF component) accepted, and under what component name/namespace? | Owner | Phase 1 |
| Q3 | Is ADR-0003 confirmed as written (C11/gnu17 port core, C++17 adapter), or amended? | Owner | Phase 1 |
| Q4 | What is the minimum supported silicon revision, and is rev < v3 in scope at all? | Owner (with hardware access) | Phase 1 (clock/PSRAM config) |
| Q5 | Which present path is primary: in-FB tile writeback vs internal-buffer + `draw_bitmap_2d` vs DMA2D hook? | Owner + Phase 2 measurements | Phase 2 |
| Q6 | What PSRAM size/type does the target board provide (affects FB count, asset residency)? | Owner (hardware) | Phase 2 |
| Q7 | Which engine (PPA vs DMA2D) owns which present-stage operation in `auto` mode? | Phase 3 measurements + owner | Phase 3 |
| Q8 | What is the declared per-channel float tolerance for float-exposed paths? | Owner (with Phase 2 data) | Phase 3 |
| Q9 | Is dual-worker determinism a hard requirement or a best-effort goal for Phase 5? | Owner | Phase 5 |
| Q10 | May `esp_private/*` APIs ever be used (e.g. DMA2D memcpy wrapper) with an explicit support-life caveat? | Owner | Phase 3 (default: no) |
| Q11 | What is the fork's versioning scheme and release channel (git tags vs registry versions)? | Owner | Phase 7 |
| Q12 | Which CI runner/build matrix (IDF versions, targets) guards the traceability gate going forward? | Owner | Phase 7 |

## 4. Standing limitations of Phase 0

- No flash, no monitor, no hardware run of any kind.
- No benchmark numbers; the comparison matrix in `04` is empty by design.
- `port/` does not exist; all "proposed" structures are design-only.
- Board/panel/touch specifics: none verified; the example's managed
  dependencies are scaffold, not supported hardware.
