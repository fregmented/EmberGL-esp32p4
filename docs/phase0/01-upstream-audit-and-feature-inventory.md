# Phase 0 — 01: Upstream Audit and Rendering Feature Inventory

Covers **deliverable 2** (upstream baseline audit) and **deliverable 4**
(rendering feature inventory). All facts below are MEASURED against this
repository at `533dc7d` (branch `develop`, tree clean) unless marked `[U]`.
Analysis date: 2026-11-26.

## 1. Provenance

| Item | Value | Evidence |
| --- | --- | --- |
| Upstream project | `github.com/EmberGL-Org/EmberGL` | `library.properties:8` (`url=`) |
| Baseline release | tag `v0.1.1` | `git tag -l` |
| Baseline commit | `cedba19604ddfa6f2de5679cf398952de7a3e554` | `git rev-parse v0.1.1` (= `git rev-parse v0.1.1^{commit}`; also `.git/packed-refs`) |
| Upstream `main` at audit | `6c197451257d3b2d800b40d4e21e5e3fe4f52ae7` | `git ls-remote` (2026-11-26, network) |
| Fork-local line | `6c19745` → `c5ac273` (agent-docs submodule) → `533dc7d` (`Update initial commits`) | `git log --oneline -5` |
| License | MIT, © 2022 Jarkko Lempiainen | `LICENSE:3`; every `src/` file header (e.g. `src/egl_core.h:4-5`) |
| Version | `0.1.1` | `VERSION`; `library.properties:2` |
| Byte-identity of `src/`, `examples/`, `LICENSE`, `library.properties` vs v0.1.1 | identical (empty diff) | `git diff --exit-code v0.1.1 -- src examples LICENSE library.properties` |
| Whole-tree delta vs v0.1.1 at `533dc7d` | 12 files, +1871/−1: FUNDING removal −1, `.gitignore` +273, `.gitmodules` +3, `CODING_CONVENTION.md` +1470, `docs/agents/general_instruction` submodule +1, `example/` +124 in 7 files | `git diff --stat v0.1.1 533dc7d` |

The fork remote is `git@github.com:fregmented/EmberGL-esp32p4.git` (a fork of
`EmberGL-org/EmberGL`). Upstream was reachable at audit time and its
`refs/tags/v0.1.1` matched the local tag hash.

## 2. Source inventory

`src/` contains **29 files, 26,162 total lines** (MEASURED:
`find src -type f | wc -l`, `wc -l src/*.h src/*.cpp src/drivers/*`).

| Area | Files | Lines |
| --- | --- | --- |
| Core / platform abstraction | `egl_core.h/.cpp`, `egl_device.h`, `egl_device_lib.h`, `egl_meta.h` | 541 + 47 + 479 + 30 + 124 = 1,221 |
| Math | `egl_math.h`, `egl_fast_math.h`, `egl_tform3.h` | 12,516 + 136 + 2,085 = 14,737 |
| Color / pixel / vertex | `egl_color.h`, `egl_pixel.h`, `egl_vertex.h` | 3,080 + 771 + 467 = 4,318 |
| Geometry / texture | `egl_mesh.h/.cpp`, `egl_texture.h/.cpp`, `egl_shaders.h` | 536 + 49 + 455 + 40 + 352 = 1,432 |
| Rasterizer | `egl_rasterizer.h/.cpp`, `egl_rasterizer_config.h`, `egl_rasterizer_target.h`, `egl_rasterizer_tiling.h/.cpp`, `egl_rasterizer_cache.h/.cpp`, `egl_rasterizer_psc.h` | 901 + 618 + 48 + 196 + 259 + 284 + 172 + 119 + 384 = 2,981 |
| Display drivers (Arduino/Teensy) | `drivers/egl_device_ili9341.h/.cpp`, `drivers/egl_device_ili9488.h/.cpp` | 486 + 540 + 247 + 200 = 1,473 |

Sum check: 1,221 + 14,737 + 4,318 + 1,432 + 2,981 + 1,473 = 26,162.

Entry point note: `library.properties:10` declares `includes=egl_device_lib.h`,
but that header conditionally includes the two Arduino drivers when `ARDUINO`
is defined and otherwise collapses to `egl_core.h`
(`src/egl_device_lib.h:15-22`). On ESP-IDF (no `ARDUINO`) it is effectively
`egl_core.h`.

## 3. Rasterizer architecture (feature inventory)

All items MEASURED from headers; line anchors given.

- **Tile-based rendering (TBR)**: 64×64 pixel tiles, morton tile order
  default; tile grid limited to < 256×256 and power-of-2 tile size asserted
  at runtime (`src/egl_rasterizer.h:87-89`,
  `src/egl_rasterizer_tiling.cpp:115,127`).
- **Hi-Z occlusion**: 16×16 Hi-Z sub-tiles inside each 64×64 tile
  (`src/egl_rasterizer_config.h:32` `rasterizer_hiz_tile_width=16`); test and
  update individually switchable (`src/egl_rasterizer_psc.h:85-86`).
- **Render targets**: up to 4 (MRT), static-asserted
  (`src/egl_rasterizer_config.h:35` `rasterizer_max_rts=4`;
  `src/egl_rasterizer.h:121` static assert; blend hooks `blend_rt0..3` in
  `src/egl_rasterizer_psc.h:132-136`).
- **Depth**: formats `none|uint8|uint16|float32`
  (`src/egl_rasterizer_target.h:29-34`); 8 depth functions
  (`src/egl_rasterizer_psc.h:44-53`); default `less_equal`, depth write on
  (`src/egl_rasterizer_psc.h:82-83`).
- **Culling**: `none|cw|ccw` triangle back-face culling, default `ccw`
  (`src/egl_rasterizer_psc.h:34-38,87`).
- **Interpolation**: `noperspective|perspective` barycentric modes, default
  perspective (`src/egl_rasterizer_psc.h:61-64,88`).
- **Blending**: exactly three fixed-function blend ops — `opaque`, `add`,
  `alpha_blend` (`src/egl_rasterizer.h:39-41`); pixel-shader-programmable
  blending beyond that.
- **Sub-pixel precision**: 4 bits (`src/egl_rasterizer_config.h:31`).
- **Commit capacities**: max 64 dispatches/commit (`pso_store_size=16384`,
  `max_pso_state_size=256`), 256 clusters + 256 cluster strips per commit,
  32 clusters/strip, 2 KB PTV buffer, vertex cache 32 KB default
  (`src/egl_rasterizer.h:85-96`).
- **DMA plumbing**: `supports_dma` device flag + `max_dma_transfers=2` +
  `dma_buffer_px = tile_w*tile_h` (`src/egl_rasterizer.h:94-96`).
- **Zero heap**: no `malloc|calloc|realloc|free|new` in `src/` outside the
  macro definitions at `src/egl_core.h:371-376`, which library code does not
  call (grep over `src/*.h|*.cpp`). All rasterizer storage is declared as
  aggregate members of `rasterizer_memory<HWDevice,Cfg>`
  (`src/egl_rasterizer.h:104-146`) — the application owns the memory.
- **Compile-time specialization**: rasterizer variants are generated by C++
  templates per feature combination; render state is `enum` constants in PSCs
  (`src/egl_rasterizer_psc.h:80-91`).

## 4. Pixel, texture, and sampler inventory

- **Pixel formats** (`src/egl_pixel.h:27-66`): `r3g3b2`, `r5g6b5`,
  `b5g6r5`, `r5g5b5a1`, `r4g4b4a4`, `r8`, `r8g8`, `r8g8b8`, `b8g8r8`,
  `r8g8b8a8`, `b8g8r8a8`, plus float-packed `r11g11b10(f)`,
  `r10g10b10a2(f)`, and 16/32-bit scalar/vector families up to
  `r32g32b32a32(f)`. No compressed formats.
- **Textures** (`src/egl_texture.h:21-68`): `.ptx` container
  (`ptx_file_ver=0x1100`), types `none|2d|3d|cube|array2d`, layout
  **linear only** (`texlayout_linear`), mip LOD count stored, filters
  `point|linear`, address `wrap|clamp`. No swizzled/tiled layouts, no
  anisotropic filtering, no trilinear mip selection.
- **Samplers**: `sampler2d<src_fmt, filter, addr>` template; immediate-mode
  texture shader `ips_texture` with position, uniform scale, and RGBA tint
  (`src/egl_shaders.h:55-77`).
- **Immediate-mode 2D**: clipped `draw_hline/vline/rect` (`int16_t`
  coordinates, `src/egl_device.h:66-71,226-278`) and unclipped
  `fast_draw_hline/vline/rect` (`uint16_t`, `src/egl_device.h:393-436`,
  tight `do{}while` loops that write the framebuffer pointer directly).
  Pixel source is an `IPShader::exec(pixel&, x, y)` — provided
  implementations include solid `ips_color_rgba32`
  (`src/egl_shaders.h:32-52`) and `ips_texture`. There are no filled-polygon,
  line-list, path, or anti-aliasing 2-D primitives beyond these.

## 5. Display drivers and device layer

- Upstream ships exactly two display drivers — ILI9341 (240×320,
  `pixfmt_b5g6r5`) and ILI9488 (320×480, `pixfmt_b5g6r5`) — both compiled
  **only** under `#ifdef ARDUINO` (`src/drivers/egl_device_ili9341.cpp:8`) and
  written against Teensy specifics (`DMAChannel`, `IMXRT_LPSPI4_S`,
  `KINETISK_SPI0`, global `s_active_dev` static + static DMA ISR,
  `src/drivers/egl_device_ili9341.cpp:51`). They are **not** reusable on
  ESP32-P4; under an IDF compile they preprocess to empty translation units.
  They remain reference material for the `HWDevice` contract only.
- The device abstraction `graphics_device<HWDevice>` declares the contract a
  P4 backend must satisfy: framebuffer pointer/format/size, `supports_dma`,
  `max_concurrency` (`src/egl_device.h:44-52`), tile-shader hook, DMA
  transfer callbacks, and the immediate-mode 2-D API.
- `max_concurrency` is a **capacity** only (`src/egl_device.h:48`): it
  multiplies per-thread buffers in `rasterizer_memory`
  (`src/egl_rasterizer.h:111,121-126,138`) and indexes `submit_tile(...,
  uint16_t thread_idx_)`. The default `rasterize_tiles()` is a plain
  single-threaded loop passing `thread_idx = 0`
  (`src/egl_rasterizer.cpp:90-103`). No upstream device overrides it; there
  is no locking anywhere in `src/`.
- **Known hazard**: `init_default_tile_shader()` keeps a function-local
  `static` shader + wrapper and rewrites `wrapper.m_dev = (HWDevice*)this` on
  every call (`src/egl_device.h:319-331`) — a cross-instance data race if two
  device instances of the same specialization are ever used. Phase 0 posture:
  single device instance only; recorded as risk R16 in `06-risks.md`.

## 6. Build/configuration model

- **Build configs**: `EGL_DEBUG` is hard-defined in the header
  (`src/egl_core.h:35`); `EGL_RELEASE`/`EGL_RETAIL` are commented out. In the
  shipped default, asserts terminate via `EGL_ABORT()` → `__builtin_trap()`
  on GCC — a device panic. Selecting a release configuration is a required
  Phase 1 decision (`06-risks.md` R15). Logging goes through `printf`
  (`EGL_LOGF` for GCC); `init_serial()` is expected at program start
  (upstream README, Development section).
- **Platform detection has no ESP32/ESP-IDF case**: `EGL_PLATFORM_*` and
  `EGL_PLATFORM_NUM_SPI/DMA` are left undefined under IDF; `EGL_MEM_ALLOC`
  maps to `malloc` (`src/egl_core.h:371-376` macro definitions, unused by
  library code).
- **No SIMD intrinsics**: grep for `_mm_|vld1|__ARM_NEON|__riscv_v|RVV` over
  `src/` returns zero hits. The library is scalar C++ with float math.
- **Syntax compatibility** (MEASURED, GCC 15.2.0 cross-compile to
  `rv32imafdc_zicsr_zifencei_zaamo_zalrsc` with `-mabi=ilp32`,
  `-fsyntax-only`; canonical command
  `riscv32-esp-elf-g++ -std=<std> -march=rv32imafdc_zicsr_zifencei_zaamo_zalrsc -mabi=ilp32 -Isrc -fsyntax-only <file>`,
  log `/tmp/opencode/embergl-p4/syntax-check.log`): a TU including all public
  headers passes at `-std=gnu++11`, `gnu++17`, `gnu++20`, and `gnu++26`; the
  six non-driver `.cpp` files pass at `-std=gnu++26`. This is a
  syntax/semantic check only — **no P4 integration build of the library
  exists in Phase 0**; the stock example does not compile or link any
  `src/` file.
- The `.ino` sketches under `examples/` are Arduino documentation artifacts
  (8 sketches with `.dae`/`.png`/`.tga` and pre-generated `p3g_*`/`ptx_*`
  headers); ESP-IDF cannot build them.

## 7. Audit conclusions for the P4 port

1. The core is genuinely platform-agnostic: no IDF-incompatible constructs
   were found; the only porting surface is the `HWDevice` device layer.
2. Zero-heap, caller-owned memory makes the P4 SRAM budget plannable
   statically (`03-platform-integration-architecture.md`).
3. Everything that touches a DMA-read surface on P4 needs cache maintenance
   the upstream code does not perform — this is the single largest behavioral
   gap (`03`, `04`).
4. The Arduino/Teensy drivers provide a pattern (tile → DMA ring → bus) but
   not reusable code.
