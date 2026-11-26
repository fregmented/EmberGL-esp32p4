---
adr-id: 0004
title: Capability-gated accelerator policies (PPA / 2-D transfer), with software twin
status: proposed
date: 2026-11-26
deciders: Phase 0 repository initialization
---

# 0004 — Capability-gated accelerator policies (PPA / 2-D transfer), with software twin

## Context

The ESP32-P4 offers hardware engines that could offload parts of a software
rendering pipeline: the Pixel Processing Accelerator (PPA) and the 2D-DMA
(DMA2D) / GDMA engines. Using them ad hoc — scattered through render paths
with implicit assumptions — would break the library's determinism, its
zero-heap memory discipline, and its software-reference correctness model.
Phase 0 verified which engines and APIs actually exist in IDF 6.1.1; nothing
is integrated yet, and no throughput has been measured.

Verified API landscape (IDF 6.1.1, `$IDF_PATH/components/...`):

- **PPA** — `esp_driver_ppa/include/driver/ppa.h`: three operations,
  `PPA_OPERATION_SRM` (scale + rotate 0/90/180/270 CCW + mirror),
  `PPA_OPERATION_BLEND` (background + foreground, A4/A8 alpha formats),
  `PPA_OPERATION_FILL` (constant pixel). Per-op pixel-format enums; SRM
  inputs include ARGB8888/RGB888/RGB565/YUV/GRAY8 (no A8 for SRM; blend adds
  A8/A4). Dimension limit: 14-bit DMA2D descriptor fields, ≤ 16383 per side
  (`hal/dma2d_ll.h:91` `DMA2D_LL_DESC_2D_FIELD_MAX 0x3FFF`); scale range
  [1/16, 256). Output buffer must be aligned to the L1 cache line
  (internal memory) or L1+L2 lines (external/PSRAM) **including** buffer
  size (`driver/ppa.h:144-145`); input may live anywhere including flash.
  Blocking or ISR-callback completion (`ppa_trans_mode_t`). The driver
  performs its own descriptor/writeback cache maintenance
  (`ppa_blend.c:280-303`).
- **Public 2-D strided copy** — `esp_async_color_convert_install_dma2d()` +
  `esp_async_color_convert()` (and `esp_color_convert_blocking()`) in
  `esp_driver_dma/include/esp_async_color_convert.h`; same-format source and
  destination means a pure 2-D copy (`:135-140`). Request carries strides,
  sub-rectangles, and fourcc formats (`:96-121`).
- **Public 1-D copy** — `esp_async_memcpy_install*` variants +
  `esp_async_memcpy()` (`esp_async_memcpy.h`); byte/word counts only, no
  stride — unsuitable for tile rows.
- **Absent names** — `esp_async_memcpy_new` and
  `esp_async_memcpy_install_gdma_ext_mem` do **not** exist in 6.1.1 (older
  documentation names; do not plan on them).
- **Private** — GDMA/2D-DMA channel management
  (`esp_private/{gdma.h,dma2d.h,gdma_link.h,dw_gdma.h}`) and the DMA2D
  2-D memcpy wrapper (`esp_private/async_memcpy_dma2d.h`) are private API.
  `esp_dma_utils.h` is effectively empty in 6.1.1.
- `esp_driver_ppa` is **not** pulled in by `esp_lcd`; using PPA requires an
  explicit component dependency (confirmed by the Phase 0 example build's
  component list, which contains `esp_driver_dma`/`esp_lcd` but not
  `esp_driver_ppa`).

## Decision (proposed — Phase 0 direction)

1. Hardware engines are used **only** through named accelerator policies,
   each with a capability gate, a policy mode (`off`/`auto`/`force`), and a
   software twin:
   - **Present-stage policy** — optional PPA use for the stage that moves
     finished pixels toward the presentable surface: tile/framebuffer
     composition fills (`FILL`), scaled/rotated full-surface presentation
     (`SRM`), alpha-blended overlay composition (`BLEND`).
   - **2-D transfer policy** — optional DMA2D use for strided 2-D pixel
     movement (tile writeback into a framebuffer, surface-to-surface
     copies) via `esp_async_color_convert`.
2. **Capability gate** (runtime, before any engine call): SoC support,
   driver installed, buffer memory class and alignment (IDF 6.1.1 has no
   public pre-allocation alignment API — derive the allocator alignment
   from the build config: `max(CONFIG_CACHE_L1_CACHE_LINE_SIZE,
   CONFIG_CACHE_L2_CACHE_LINE_SIZE)` for external memory,
   `CONFIG_CACHE_L1_CACHE_LINE_SIZE` for internal-only; after allocation,
   verify `esp_cache_get_line_size_by_addr(buf)` is non-zero and divides
   both address and size; never hardcode 64 B; MSPI/DMA2D alignment may
   exceed the cache line), dimension ≤ 16383, format membership in the
   op's format list, in-flight buffer exclusivity.
3. **Policy modes**: `off` = software twin always; `auto` = hardware when
   the gate passes, otherwise the software twin; `force` = hardware or an
   **explicit, returned failure** — never a silent fallback to software.
4. **No accelerator code in `src/`.** `src/` stays frozen; policies live in
   the platform port (ADR-0002/0003 boundary).
5. No private IDF APIs (`esp_private/*`) in anything documented as public
   contract; if `esp_private/async_memcpy_dma2d.h` is ever wanted, that is a
   new ADR.
6. **No performance claims.** Engine selection in Phase 0/1 is justified by
   functional fit and measurement discipline (see `docs/phase0/04`), never
   by assumed speedups.

## Alternatives considered

Using no hardware engines at all was rejected because it abandons
demonstrated SoC capabilities without measurement; making PPA/DMA2D
mandatory with no software fallback was rejected because it couples
correctness to engine availability and alignment envelopes that vary by
configuration and memory class.

## Consequences

- Determinism and correctness have a reference implementation: every
  accelerated path has an executable software twin to diff against.
- `force` failing loudly prevents "it worked on this board" asymmetries
  between boards with/without PSRAM or engines.
- Alignment/dimension checks add per-request cost but remove a class of
  silent corruption (misaligned PPA output under PSRAM + ECC constraints).
- Phase 1+ must own the real cost question (queue depth, `max_pending_trans_num`,
  ISR callback overhead) with measurements; none exist today.

## Evidence

- MEASURED (headers): `driver/ppa.h:20-25` ops, `:63` `ppa_register_client`,
  `:144-145` output alignment note, `:163-166` modes, `:209/:270/:302`
  `ppa_do_scale_rotate_mirror`/`ppa_do_blend`/`ppa_do_fill`;
  `hal/ppa_types.h:30-96` formats/alpha/rotation;
  `hal/dma2d_ll.h:91` `DMA2D_LL_DESC_2D_FIELD_MAX 0x3FFF`;
  `ppa_srm.c:216-253` flash/limit checks; `ppa_core.c:596-606` alignment
  enforcement; `ppa_blend.c:280-303` `esp_cache_msync` around transfers.
- MEASURED (headers): `esp_async_color_convert.h:96-121` request,
  `:135-140` same-format ⇒ 2-D copy, `:157-191` API + blocking variant;
  `esp_async_memcpy.h:62-70,103-232` 1-D API set; absence of
  `esp_async_memcpy_new`/`esp_async_memcpy_install_gdma_ext_mem` verified by
  header inspection; `esp_private/async_memcpy_dma2d.h:8` marks the DMA2D
  memcpy wrapper private.
- MEASURED (build): Phase 0 example build component list contains
  `esp_driver_dma`, `esp_lcd`, `esp_mm` and does **not** contain
  `esp_driver_ppa` (`/tmp/opencode/embergl-p4/example-set-target.log`).
- UNVERIFIED `[U]`: any throughput, latency, or bandwidth number for PPA or
  DMA2D on ESP32-P4 — nothing measured in Phase 0.
