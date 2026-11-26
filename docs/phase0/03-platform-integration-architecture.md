# Phase 0 — 03: Platform Integration Architecture and Memory/Execution Proposal

Covers **deliverable 5** (platform integration architecture) and
**deliverable 7** (memory and execution model proposal). This document is a
design proposal: nothing in it is implemented, and no Phase 1 symbol names
are fixed here. Vocabulary per `GLOSSARY.md`; constraints per ADR-0001/0002/
0003/0004.

## 1. Proposed repository layout

```text
/                      repo root
├── src/               EmberGL core — FROZEN upstream snapshot (ADR-0001)
├── examples/          upstream Arduino sketches — reference only, untouched
├── doc/               upstream docs — untouched
├── port/              (NOT created in Phase 0 — proposed Phase 1)
│   ├── c/             repo-owned platform core: C11 subset, -std=gnu17, MISRA C:2023 (ADR-0003)
│   └── cpp/           thin C++17 adapter: graphics_device<HWDevice> specializations, PSC glue
├── example/           stock ESP32-P4 test project (build-verification target)
└── docs/              ADRs, agent docs, phase0 design docs
```

The root becomes an ESP-IDF component (`embergl`) only if ADR-0002 is
accepted and implemented in Phase 1; its files do not exist today.

## 2. Dependency direction

```text
examples (Arduino, upstream) ─┐
                              ├─► src/ (frozen core, template-driven)
port/cpp (C++17 adapter) ─────┤
                              │
port/c (C platform core) ─────┘──► ESP-IDF components (esp_mm, esp_lcd,
                                    esp_driver_ppa, esp_driver_dma, heap, freertos)
```

Rules:

- Arrows point **down**: `src/` never includes IDF headers or `port/` code
  (ADR-0003 §5). Platform knowledge enters only through the `HWDevice`
  contract (`src/egl_device.h:44-52` and its virtual hooks).
- `port/cpp` depends on `src/` (templates) and `port/c` (C ABI). `port/c`
  never includes `src/` headers except through the adapter where a pure
  value-type is required — and then only header-only, definition-free types.
- IDF components are referenced only by `port/c` (and by the example
  project), never by `src/`.

## 3. Data contracts

Four boundaries carry all information across the `extern "C"` line
(ADR-0003 §4). Shapes below are structural sketches, not symbol names.

### 3.1 Arena contract (caller-provided arena)

- The caller owns all memory, as upstream does (`rasterizer_memory`,
  `src/egl_rasterizer.h:104-146`): zero steady-state allocation.
- The port asks the caller for a described arena: one descriptor carrying
  base pointer, size, memory class (internal SRAM vs PSRAM), and the
  allocator alignment, derived from the build config
  (`max(CONFIG_CACHE_L1_CACHE_LINE_SIZE, CONFIG_CACHE_L2_CACHE_LINE_SIZE)`
  for external memory, `CONFIG_CACHE_L1_CACHE_LINE_SIZE` for internal-only)
  and verified post-allocation with the public
  `esp_cache_get_line_size_by_addr(buf)` (non-zero, divides address and
  size) — never hardcoded (see `02` row 5).
- Sub-allocations (tile buffer, depth, Hi-Z, vcache, PSO store, DMA ring)
  are carved deterministically by offset; the carve table is a stable,
  documented layout so two builds of the same config have identical
  footprints.

### 3.2 Framebuffer contract

- The framebuffer is **caller- or driver-owned**: on the DPI path the panel
  driver allocates 1–3 PSRAM framebuffers (`esp_lcd_panel_dpi.c:226-245`);
  the port can either render into the driver's FB directly (preferred fast
  path, see §5) or into a caller-provided PSRAM surface.
- The contract records: base pointer, stride, pixel format (EmberGL
  `e_pixel_format` ↔ IDF fourcc mapping table maintained in one place),
  alignment, and ownership (who msyncs, who presents).
- The mapping must state byte order explicitly: DPI `RGB888` is fourcc
  `BGR24` (byte order B,G,R); EmberGL `pixfmt_b5g6r5` vs `pixfmt_r5g6b5`
  must be matched against `ESP_COLOR_FOURCC_RGB16`'s little-endian layout and
  **verified on hardware `[U]`** before any claim.

### 3.3 Capability contract (capability gate, ADR-0004)

- One query surface reports, per boot: engine availability (PPA installed?
  DMA2D channel available?), cache line sizes per memory region, max
  transfer dimensions, supported format lists, PSRAM presence/size.
- The port consumes only the capability report; render paths never test
  `#ifdef`-style platform facts at their call sites.

### 3.4 Policy contract (policy mode, ADR-0004)

- Each accelerator policy (present-stage, 2-D transfer) is configured with a
  mode: `off` / `auto` / `force`. `force` fails with an explicit error; it
  never silently falls back to the software twin.

### 3.5 Context contract (rendering context)

- One rendering context = one device + rasterizer + its buffers. Phase 0/1
  support exactly one context. A context descriptor groups the arena
  descriptor, framebuffer contract, capability report, and policy
  configuration.

## 4. Memory model (proposal)

- **Internal SRAM** (ESTIMATED ~620 KB usable on rev ≥ v3 silicon: 768 KB
  window − 128 KB default L2 carve-out − ~20 KB top reservation; rev < v3
  gives ≈236 KiB — see `02` row 3):
  render tile (64×64), depth tile, Hi-Z, PTV buffer, per-worker tile
  buffers, stacks, port statics. Illustrative footprint for a 480×800
  RGB565 device with default config, `max_concurrency=1` (ESTIMATED from the
  `rasterizer_memory` layout, `src/egl_rasterizer.h:104-146` — arithmetic,
  **not measured**): rtile0 8 KB + depth 8 KB + hi-z 32 B + PTV 2 KB +
  vcache 32 KB + PSO store 16 KB + dispatches 1 KB + tile map/tiles +
  clusters/strips + 2×4 Kpx DMA buffer (16 KB at RGB565) ≈ **~90 KB** before
  tiles/strips arrays; the full ordering must be recomputed per config in
  Phase 1.
- **PSRAM**: framebuffer(s) and large read-only assets (`.ptx` textures,
  `.p3g` geometry blobs — both are load-in-place formats upstream).
  Requires explicit `CONFIG_SPIRAM=y` (default off, `02` row 7). Anything a
  DMA/PPA engine touches must be allocated with
  `heap_caps_aligned_calloc(<runtime alignment>, 1, size,
  MALLOC_CAP_SPIRAM|MALLOC_CAP_8BIT|MALLOC_CAP_DMA)` and cache-managed.
- **Cache discipline** (normative, from AGENTS.md): C2M writeback before any
  device reads a CPU-written buffer; M2C invalidate after a device writes a
  CPU-read buffer; no buffer reuse while a transfer is in flight. Internal
  SRAM is L1-cached on P4 — the same rules apply.
- No allocations after init (CODING_CONVENTION §5.2); all buffers have a
  single documented owner.

## 5. Execution model (proposal)

- **Frame loop** (single context, single worker):
  1. `dispatch_pso()` — cluster setup/binning (CPU, upstream code).
  2. `commit()` — tiles rasterized one at a time into internal tile
     buffers; completed tile transferred to the presentable surface.
  3. Present: either (a) tile writeback directly into the driver-owned DPI
     FB region + `esp_lcd_panel_draw_bitmap()` on the touched rect — the
     driver then only msyncs affected rows, no copy
     (`esp_lcd_panel_dpi.c:640-668`); or (b) tile → internal DMA buffer →
     `esp_lcd_panel_draw_bitmap_2d()`; or (c) `esp_lcd_dpi_panel_enable_dma2d()`
     hook. Choice is part of the present-stage/2-D transfer policies.
- **Async completion**: engine work uses ISR-callback completion
  (`ppa_trans_mode_t`, `esp_async_*` callbacks); the render loop either
  double-buffers the tile buffer slots (upstream DMA ring is
  `max_dma_transfers=2`, `src/egl_rasterizer.h:94-96`) or blocks — decided
  in Phase 1 with measurements.
- **Workers** (proposed Phase 5, not earlier): ≤ cores−1 = 1 extra worker;
  disjoint tile subsets; per-`thread_idx` buffers (upstream indexes them
  already); shared vcache requires locking (none exists upstream);
  deterministic mode guarantees identical output regardless of worker count.
  The static default-tile-shader wrapper (`src/egl_device.h:319-331`) must be
  resolved before any second device instance.

## 6. Failure and fallback

- Every engine call goes through the capability gate; `auto` falls back to
  the software twin on gate failure, `force` returns an error instead.
- The software twin is the reference for correctness diffs: each accelerated
  operation must produce results within the declared tolerance of its twin
  (integer paths exact; see `04-benchmark-methodology.md`).
- Timeout policy for blocking engine waits (CODING_CONVENTION §14); no
  unbounded retries; ISR handlers short and bounded (§15).
- Cache/DMA misuse classes (stale cache, in-flight reuse, misalignment) are
  prevented by contract checks at the boundary, not by convention alone.

## 7. Compatibility boundaries

- **EmberGL API**: no public upstream API changes; the port adds devices,
  not APIs. PSC/user-shader interfaces remain template C++ used from
  `port/cpp` or user code.
- **Upstream behavior**: rasterization results must match the upstream
  algorithm exactly for identical inputs and configuration (same
  top-left rule, same sub-pixel precision) — the software twin also guards
  this.
- **IDF**: validated against 6.1.1 only; other 6.1.x versions are expected
  compatible `[U]`, other majors unverified.
- **Arduino upstream layout**: untouched (ADR-0002 §Decision 3).
- **License**: MIT notice preservation (ADR-0001 §Decision 2).
