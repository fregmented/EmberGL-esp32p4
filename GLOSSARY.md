# GLOSSARY

Single context: **EmberGL ESP32-P4 port**. These terms are canonical for all
docs, issues, ADRs, and future code in this repository. When your output names
one of these concepts, use the term as defined here.

Terms marked *(design)* describe planned Phase 1+ structure; they must not be
described as existing until implemented.

## Baseline and traceability

**EmberGL core**: the upstream tile-based software rasterizer library that
this fork specializes — headers, sources, math, and rasterizer under `src/`.
_Avoid_: "the library code" ambiguously including port code; "EmberGL" alone
when you mean the fork.

**Upstream baseline**: the exact upstream release this fork starts from:
EmberGL v0.1.1, tag commit `cedba19604ddfa6f2de5679cf398952de3a554d`, MIT,
© 2022 Jarkko Lempiainen, at `github.com/EmberGL-Org/EmberGL`. _Avoid_: "main"
or "HEAD" as baseline references; vague "current upstream".

**Frozen snapshot**: the invariant that `src/`, `examples/`, and `doc/` are
byte-identical to the upstream baseline. Enforced by the traceability diff
gate; only an ADR-0001 ledger entry with owner approval can change it.
_Avoid_: "protected files" without naming the paths; "we can patch it later"
casually.

**Patch ledger**: *(design)* the ADR-0001 record of every permitted deviation
from the frozen snapshot — one entry per deviation, with patch ID, baseline
hash, affected paths/symbols, change, rationale, linked ADR/issue,
verification evidence, and upstream status. Empty in Phase 0. _Avoid_: keeping
patch knowledge in commit messages alone.

**Traceability diff gate**: the two commands that prove the frozen snapshot:
`git diff --exit-code v0.1.1 -- src examples doc` and
`git diff --exit-code v0.1.1 -- src examples LICENSE library.properties`.
Both must exit 0 on every Phase 0 commit and PR. _Avoid_: substituting
`--stat` or eyeballing.

## Specialization and platform layers

**ESP32-P4 specialization**: this fork's purpose — adapting the EmberGL core
to the ESP32-P4 SoC: its memory map, caches, DMA/2D-DMA/PPA engines, ISA,
and FreeRTOS environment. Scoped to the SoC, independent of any board.
_Avoid_: calling the port a "driver" for a display.

**Board/display/product specialization**: adaptation to a concrete carrier
board, panel model, touch controller, or end product. Explicitly out of scope
for `src/` and `port/`; belongs to applications and examples. _Avoid_: letting
board pinouts, panel init sequences, or product names leak into repo-owned
platform code.

**Platform port**: *(design)* repo-owned platform code in `port/c/` — the C
layer that implements device/adapter functionality on ESP-IDF. Not created in
Phase 0. _Avoid_: "port" as a verb for modifying `src/`.

**Platform adapter**: *(design)* the thin C++17 layer in `port/cpp/` bridging
the template-heavy EmberGL core (`graphics_device`, PSCs) to the C platform
port. Not created in Phase 0. _Avoid_: moving template code into `port/c/`.

**P4 backend**: *(design)* the ESP32-P4 implementation behind the upstream
`HWDevice` interface (`src/egl_device.h`) — framebuffer ownership, tile
transfer, present path. Not implemented in Phase 0. _Avoid_: claiming the
upstream ILI9341/ILI9488 drivers as "the backend" — they are
Arduino/Teensy-only reference code.

**Device adapter**: *(design)* the concrete realization of the upstream
`HWDevice` contract (framebuffer pointer, format, `supports_dma`,
`max_concurrency`, DMA transfer callbacks) by the P4 backend. _Avoid_:
inventing upstream device APIs that do not exist.

## Acceleration policies

**Software twin**: *(design)* the pure-CPU implementation of each accelerated
operation, kept as the functional reference and mandatory fallback. The
software path must produce the defined result for every input the hardware
path accepts. _Avoid_: treating the accelerator as the only implementation.

**Accelerator policy**: *(design)* a declared, capability-gated choice of
whether/how a hardware engine (PPA, 2D-DMA, GDMA) may perform a class of
work. Includes the capability gate, the policy mode, and the software twin.
_Avoid_: hardwiring engine use into render paths.

**Present-stage policy**: *(design)* the accelerator policy governing the
stage that moves finished pixels to the presentable surface (tile/fb
composition, scaling, rotation, blend of layers) — e.g. PPA SRM/BLEND/FILL
candidates. _Avoid_: calling per-tile rendering "present".

**2-D transfer policy**: *(design)* the accelerator policy governing strided
2-D pixel movement (tile writeback, surface-to-surface copies) — e.g.
`esp_async_color_convert` (DMA2D) candidates. _Avoid_: using 1-D
`esp_async_memcpy` for strided tile data.

**Capability gate**: *(design)* the runtime check that decides whether an
accelerator can serve a request: SoC feature, driver availability, buffer
location/alignment, dimension limits, format support. _Avoid_: compile-time
assumptions about alignment or limits.

**Policy mode**: *(design)* the per-policy selector: `off` (software twin
always), `auto` (hardware when the capability gate passes, else software),
`force` (hardware or explicit failure — never a silent fallback). _Avoid_:
silent degradation in `force` mode; undocumented default modes.

## Memory model

**Caller-provided arena**: the memory discipline inherited from upstream —
all rasterizer/context storage is declared and owned by the caller (no
steady-state heap allocation; zero `malloc`/`new` in the core). _Avoid_:
introducing hidden allocations in port code.

**Tile buffer**: the internal-SRAM working memory holding the 64×64-pixel
render tile (plus depth/Hi-Z) during rasterization. Sits in `rasterizer_memory`
(`src/egl_rasterizer.h:104-146`), sized by the rasterizer config. _Avoid_:
confusing with the DMA transfer buffer.

**Framebuffer**: the full-frame pixel surface to be presented. On the P4
display path it is expected to live in PSRAM (DPI panel drivers allocate it
from `MALLOC_CAP_SPIRAM`). Phase 0 does not implement or measure this.
_Avoid_: assuming framebuffer memory is internal SRAM; assuming it is
cache-coherent.

**Rendering context**: one independent rendering state machine (device +
rasterizer + buffers). Phase 0 assumes exactly one. _Avoid_: designing APIs
that implicitly share context state.

**Worker**: *(design)* a thread executing rasterization for a disjoint subset
of tiles. Upstream supports capacity only (`max_concurrency`,
`src/egl_device.h:48`) — the default `rasterize_tiles()` is single-threaded
with `thread_idx` always 0 (`src/egl_rasterizer.cpp:90-103`). Not implemented
in Phase 0. _Avoid_: claiming multithreaded rendering today.

**Deterministic mode**: *(design)* a mode guaranteeing bit-identical output
regardless of worker count or scheduling. Design goal for Phase 5.
_Avoid_: presenting floating-point results as automatically reproducible.

**Vertex cache**: the post-transform vertex cache (`vcache_size=32768`
default, `src/egl_rasterizer.h:93`) reused across tiles. If workers are ever
added, it becomes shared state requiring locking. _Avoid_: assuming lock-free
sharing.

## Targets and evidence

**Stock P4 example**: the IDF scaffold in `example/` — the build-verification
target for `esp32p4`. It is a hello_world template with managed display/touch
component dependencies; it does not link or exercise EmberGL in Phase 0.
_Avoid_: presenting it as an EmberGL demo; citing its build as library
validation.

**Evidence label**: the required classification of factual claims:
MEASURED (executed run), ESTIMATED (derived arithmetic from verified facts),
UNVERIFIED (marked `[U]` inline). See AGENTS.md "Evidence and reporting".
_Avoid_: unstated provenance; numbers without labels.
