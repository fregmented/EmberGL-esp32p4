# AGENTS.md — EmberGL ESP32-P4 fork

Operating instructions for autonomous agents working in this repository.

## Project scope

This repository is the ESP32-P4 port of EmberGL. EmberGL is a general-purpose,
low-level 2D/3D software rasterization library for memory-constrained systems.

- The specialization target is the **SoC**: ESP32-P4 (SoC-level capabilities:
  memory map, cache, DMA/PPA engines, ISA, FreeRTOS). It is never a
  board/display/product/application specialization. This fork must remain a
  general-purpose library: anything tied to a specific display, panel, touch
  controller, carrier board, or end product is out of scope for `src/` and the
  future `port/` layer.
- The upstream baseline is EmberGL **v0.1.1** (`cedba19604ddfa6f2de5679cf398952de7a3e554`,
  MIT, © 2022 Jarkko Lempiainen). `src/`, `examples/`, `doc/` are frozen
  upstream code (see ADR-0001).
- **Phase 0 is design only.** No platform code, no component packaging, no
  hardware integration exists yet. Docs must not describe planned work as
  existing. The executed **build** verification in Phase 0 is the stock
  `example/` project build for `esp32p4` plus `-fsyntax-only`
  cross-compiles of the frozen `src/` — neither links nor runs EmberGL.
  Git traceability gates and GitHub tracker commands were also executed,
  but they are process checks, not product verification.

## Read first and precedence

Before making any change, read in this order:

1. `docs/agents/general_instruction/*` (submodule; the general agent
   instructions, including `git_workflow.md` and `readme_authoring.md`).
2. `CODING_CONVENTION.md` — the project coding standard (rule hierarchy,
   MISRA C:2023 baseline, agent workflow).
3. `docs/agents/domain.md` — how to consume domain docs.
4. `GLOSSARY.md` — canonical vocabulary; use these terms.
5. Relevant ADRs in `docs/adr/`.
6. Task-specific docs (`docs/phase0/`).

Precedence when documents conflict: the explicit task brief > ADRs >
`AGENTS.md` > `docs/agents/*` > `GLOSSARY.md`, with one carve-out: for
**code style and safety rules, `CODING_CONVENTION.md` outranks ADRs and
every other document for repo-owned code** — its §1 rule hierarchy is
authoritative for anything this repository owns. `CODING_CONVENTION.md`
explicitly exempts third-party/vendor code from conformance; upstream
`src/` is third-party code.

## Toolchain and build verification

- Activate the toolchain in every shell before running IDF commands:

  ```bash
  source /home/hanwool/.espressif/tools/activate_idf_v6.1.1.sh
  ```

- Versions: ESP-IDF **v6.1.1** (`~/.espressif/v6.1.1/esp-idf`), GCC
  **15.2.0** (`riscv32-esp-elf-gcc`, crosstool-NG `esp-15.2.0_20251204`),
  Python 3.12.3. Build target: `esp32p4`.
- `example/` is the ESP32-P4 test project (a stock IDF scaffold with managed
  display/touch component dependencies pinned in `example/main/idf_component.yml`).
  It is the build-verification target only; it does not yet use EmberGL.
- **Phase 0 artifact-hygiene rule: do not run `idf.py` inside the
  repository.** Reason: `idf.py set-target` runs `fullclean` and writes
  `sdkconfig`, `dependencies.lock`, `managed_components/`, and `build/`
  into the project directory (observed in the Phase 0 run:
  `/tmp/opencode/embergl-p4/example-set-target.log`), which would dirty the
  frozen tree. Later phases may build in-tree once these artifacts are
  properly ignored — but `dependencies.lock` is deliberately **not**
  ignored, so it must then be committed deliberately. Until then, build
  only from a copy:

  ```bash
  cd <repository root>   # the repo itself; no build happens here
  REPO="$PWD"
  WORK=/tmp/opencode/embergl-p4
  rm -rf "$WORK/example-build-src" "$WORK/example-build"
  cp -a "$REPO/example" "$WORK/example-build-src"
  source /home/hanwool/.espressif/tools/activate_idf_v6.1.1.sh
  cd "$WORK/example-build-src"
  idf.py set-target esp32p4
  idf.py -B "$WORK/example-build" build
  ```

  PASS requires both commands to return 0. Record exit codes and the count of
  lines matching `warning:` in the build log.
- Flashing (`idf.py flash`) and serial monitoring (`idf.py monitor`) require
  explicit human approval. Do not claim hardware validation without a
  hardware run.

## Repository boundaries

- **Frozen upstream paths** — never edit: `src/`, `examples/`, `doc/`.
  Also never edit `LICENSE`, `VERSION`, `library.properties` (upstream
  metadata), or the `docs/agents/general_instruction` submodule.
- Verification gate (traceability, from ADR-0001):

  ```bash
  git diff --exit-code v0.1.1 -- src examples doc
  git diff --exit-code v0.1.1 -- src examples LICENSE library.properties
  ```

- Repo-owned Phase 0 additions live in: `AGENTS.md`, `GLOSSARY.md`,
  `.gitignore`, `docs/adr/`, `docs/agents/*.md` (not the submodule),
  `docs/phase0/`, `README.md` (prefix-only extension).
  `CODING_CONVENTION.md` is also repo-owned (committed just before Phase 0);
  edits to it follow its own §1 rule hierarchy.
- Do not create in Phase 0: root `CMakeLists.txt`, root `idf_component.yml`,
  `port/`, `CLAUDE.md`, `GLOSSARY-MAP.md`. The root-as-component proposal is
  ADR-0002 (Phase 1); its files do not exist yet.
- `examples/` (plural) holds upstream Arduino sketches — reference material
  only; they cannot be built by ESP-IDF.

## Language and safety boundary

The language/safety split is decided in ADR-0003 (proposed; to be confirmed in
Phase 1). The Phase 0 direction:

- Upstream C++ (`src/`) is **frozen third-party code**: pre-C++11 style plus
  `static_assert`, header-heavy templates, compiles unmodified under the IDF
  `-std=gnu++26` default. Never modernize or reformat it.
- Repo-owned platform code goes to `port/c/` as **C** (C11-compatible subset,
  built with `-std=gnu17`, conforming to MISRA C:2023 per
  `CODING_CONVENTION.md`).
- A thin C++17 adapter in `port/cpp/` bridges template-heavy upstream types
  (e.g. `graphics_device` subclasses, PSCs) to the C core. Template interop
  stays in the adapter; it must not leak into `port/c/`.
- Every crossing of the C boundary uses `extern "C"` linkage; no C++ types,
  exceptions, or RTTI cross the ABI.
- `src/egl_*.h` must never include ESP-IDF headers. Platform knowledge enters
  only through the `HWDevice` interface (`src/egl_device.h`) implemented in
  `port/`.
- `EGL_DEBUG` is hard-defined at `src/egl_core.h:35` (debug build = asserts +
  `__builtin_trap()`); the release/retail build selection is a required
  Phase 1 decision (see `docs/phase0/06`).
- ADR-0003 is a Phase 1 confirmation item, not settled law.

## Memory, DMA, and cache rules

Facts verified for ESP32-P4 under IDF 6.1.1 (evidence in `docs/phase0/02`):

- Internal SRAM window is 768 KB (`SOC_IRAM_LOW 0x4ff00000` …
  `SOC_IRAM_HIGH 0x4ffc0000`). Usable internal DRAM is an **ESTIMATED
  ~620 KB** for rev ≥ v3 silicon: 768 KB − 128 KB default L2 carve-out
  (`CONFIG_CACHE_L2_CACHE_SIZE=0x20000`) − ~20 KB top reservation
  (`0x5040`; usable heap ends at `0x4ffbafc0`). If the target is rev < v3
  (silicon revision `[U]`), the layout differs: usable end `0x4ff3afc0`,
  ≈236 KiB — detect the silicon revision at runtime, never assume one
  layout. The carve-out is already
  subtracted in this arithmetic; do not subtract it twice.
- Memory class is caller/provider-selected, not fixed by this project.
  The Phase 0 design default places tile buffers in internal SRAM and
  framebuffers in PSRAM **where PSRAM exists**; tile-buffer placement in
  internal SRAM is an unvalidated assumption (`[U]`, `docs/phase0/06`
  A3) pending Phase 1/2 measurement. Nothing in the port may require
  PSRAM to be present.
- `CONFIG_SPIRAM` defaults to **off**; the PSRAM aperture is 64 MB at
  `0x48000000` (`SOC_EXTRAM_LOW..SOC_EXTRAM_HIGH`), but on-board capacity is
  runtime-detected and unverified for any board.
- Cache lines: L1 = 64 B (`CONFIG_CACHE_L1_CACHE_LINE_SIZE=64`), L2 = 64 B
  default (128/256/512 KB selectable; 512 KB forces 128 B lines).
  **Never hardcode alignment.** IDF 6.1.1 has no public pre-allocation
  alignment API: `esp_cache_get_alignment()` and `dma2d_get_alloc_alignment()`
  are private (`esp_private/esp_cache_private.h`, `esp_private/async_memcpy_dma2d.h`),
  and the public `esp_cache_get_line_size_by_addr(addr)` requires an existing
  address (returns 0 for NULL/non-cacheable). Recipe: derive the allocator
  alignment from the build config — `max(CONFIG_CACHE_L1_CACHE_LINE_SIZE,
  CONFIG_CACHE_L2_CACHE_LINE_SIZE)` for external (PSRAM) memory,
  `CONFIG_CACHE_L1_CACHE_LINE_SIZE` for internal-only buffers — then, after
  allocation, verify `esp_cache_get_line_size_by_addr(buf)` is non-zero and
  divides both the address and the size. MSPI/DMA2D alignment may exceed the
  cache line; relying on private helpers is a separate decision (see
  `docs/phase0/06` Q10).
- **Internal SRAM is L1-cached** on P4 (`SOC_CACHE_INTERNAL_MEM_VIA_L1CACHE 1`).
  There is no coherent internal RAM: every CPU-written buffer that DMA/PPA
  reads needs `esp_cache_msync(buf, size, ESP_CACHE_MSYNC_FLAG_DIR_C2M)` before
  the transfer, and every DMA-written buffer the CPU reads needs
  `DIR_M2C` afterwards.
- The cache maintenance model (`$IDF_PATH/components/esp_mm/include/esp_cache.h`):
  C2M = writeback CPU→memory before a device reads; M2C = invalidate
  memory→CPU after a device writes. Never reuse or free a buffer while a
  DMA/PPA transfer may still touch it (`CODING_CONVENTION.md` §18).
- The upstream `fast_draw_hline/vline/rect` paths write directly to the
  framebuffer pointer (`src/egl_device.h:393-436`) with no cache awareness:
  after any `fast_draw_*` into a DMA-read surface, a C2M msync of the touched
  rows is mandatory before device consumption.
- Allocation recipe for any buffer devices will touch:
  `heap_caps_aligned_calloc(<config-derived alignment — see the cache-lines
  bullet above>, 1, size, MALLOC_CAP_SPIRAM|MALLOC_CAP_8BIT|MALLOC_CAP_DMA)`
  for PSRAM buffers (pattern used by
  `$IDF_PATH/components/esp_lcd/dsi/esp_lcd_panel_dpi.c:233-245`, which
  uses the driver-internal alignment helper).
- EmberGL's own memory model is zero-heap: all rasterizer storage is declared
  by the app (`rasterizer_memory` aggregate, `src/egl_rasterizer.h:104-146`).
  Keep it that way: caller-provided arena, no steady-state allocation.
- No private IDF APIs (`esp_private/*`) may be depended on by anything
  documented as a public contract. No accelerator code in `src/`.

## Concurrency and determinism

- Default model: **one rendering context, one worker**. Upstream
  `max_concurrency` is a *capacity* only (`src/egl_device.h:48`), and the
  stock `rasterize_tiles()` runs single-threaded with `thread_idx` always 0
  (`src/egl_rasterizer.cpp:90-103`). Multicore is not implemented upstream.
- If worker parallelism is ever enabled (proposed Phase 5): workers ≤ cores−1
  (cores = 2 on P4, `SOC_CPU_CORES_NUM`), tiles partitioned into **disjoint**
  subsets per worker, buffers indexed per `thread_idx`, and any shared
  structure (notably the vertex cache) explicitly locked. There is no locking
  in upstream code today.
- Determinism requirement: same input + same configuration ⇒ same pixels.
  Parallel tile rendering must not change results vs. the single-worker
  reference. A deterministic mode is a design goal for Phase 5, not a
  current feature.
- Known cross-instance hazard: `init_default_tile_shader()` stores a
  function-local `static` wrapper and rewrites its device pointer on every
  call (`src/egl_device.h:319-331`); two device instances of the same type
  race. Do not create multiple concurrent device instances of the same
  specialization until this is addressed.
- FreeRTOS on P4 is SMP (2 cores default), `xTaskCreatePinnedToCore` and
  `esp_ipc_call*` exist, `portMUX_TYPE` is a spinlock — verified in
  `docs/phase0/02`. No concurrency model beyond the above is verified.

## Git workflow

Follow `docs/agents/general_instruction/git_workflow.md`:

- Branch from a clean `develop`: `git checkout -b <type>/<topic> develop`.
  One logical change per commit; descriptive subject + body explaining what
  and why; end the body with a `Model:` trailer naming the model that did the
  work (a real value, never a placeholder).
- Never `git push --force`, `reset`, `clean`, `rebase`, or `amend` published
  commits.
- PRs target `develop`. Agents open PRs but **do not merge** them.
- Before every commit: `git diff --cached --check` (no whitespace errors),
  and re-run the protected-path gates above.

## Evidence and reporting

Every factual claim in docs, commits, and reports is labeled:

- **MEASURED** — observed in an actually executed command/run on the stated
  environment. Cite the command and artifact (log path, exit code).
- **ESTIMATED** — derived by computation from verified facts (e.g. buffer
  footprint arithmetic). Show the derivation.
- **UNVERIFIED** — believed but not checked, or checked only on paper. Mark
  inline with `[U]` in docs. An unverified fact must never be restated
  elsewhere without the `[U]` marker.

Rules:

- Never claim hardware validation, flash/monitor results, or performance
  numbers without the corresponding run. Phase 0 has **no** measured
  performance numbers, by design.
- Phase 0 "build PASS" means host cross-compilation of the stock example
  succeeded — it does **not** mean EmberGL was linked or exercised.
- A final report must state: files changed, commits, commands run,
  PASS/FAIL/NOT RUN per check, warning counts, limitations, and open
  questions.

## Definition of Done

A change is done when all applicable items hold:

- [ ] Scope matches the task brief; nothing extra was created or edited.
- [ ] Protected paths untouched (`git diff --exit-code v0.1.1 -- src examples doc`).
- [ ] New/changed C code would follow `CODING_CONVENTION.md` (MISRA C:2023
      baseline for repo-owned code).
- [ ] Docs use glossary vocabulary and mark unverified facts `[U]`.
- [ ] Build verification (in /tmp copy) PASS with exit codes recorded, if
      anything build-relevant changed.
- [ ] `git diff --cached --check` clean before each commit.
- [ ] Commits are logical units with `Model:` trailers.
- [ ] PR opened against `develop`; not merged by the agent.
- [ ] Final report lists files, verification results, limitations, `[U]` items.

## Agent skills

### Issue tracker

Issues are tracked on GitHub for `fregmented/EmberGL-esp32p4` (always pin the
repo with `-R fregmented/EmberGL-esp32p4`; this is a fork — never file into
the upstream parent). See `docs/agents/issue-tracker.md`.

### Triage labels

Five default triage labels: `needs-triage`, `needs-info`, `ready-for-agent`,
`ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context layout: one `GLOSSARY.md` at the repo root plus `docs/adr/`.
Read `GLOSSARY.md` and the relevant ADRs before domain work. See
`docs/agents/domain.md`.
