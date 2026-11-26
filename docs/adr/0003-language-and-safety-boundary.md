---
adr-id: 0003
title: Language and safety boundary (C port core, C++ adapter)
status: proposed
date: 2026-11-26
deciders: Phase 0 repository initialization
---

# 0003 — Language and safety boundary (C port core, C++ adapter)

## Context

EmberGL upstream is C++ (template-heavy, pre-C++11 style plus
`static_assert`; verified to compile unmodified under IDF's `-std=gnu++26`
with `-fno-exceptions -fno-rtti`). `CODING_CONVENTION.md` mandates C (C11
subset) and MISRA C:2023 for **project-owned** code, and explicitly exempts
third-party/vendor code. A port needs code on both sides of that line:

- frozen upstream C++ (`src/`) — third-party, exempt;
- repo-owned platform code (device adapter, cache/DMA plumbing, policies) —
  must satisfy the convention.

Phase 0 must pick the boundary before any `port/` code exists, because the
boundary determines the public ABI for everything Phase 1+ builds.

## Decision (proposed — Phase 1 confirmation item)

1. **Repo-owned platform code is C.** New platform code lives in `port/c/`,
   written in a C11-compatible subset compiled with `-std=gnu17`, conforming
   to `CODING_CONVENTION.md` and MISRA C:2023 (mandatory rules without
   exception; required rules need documented deviations).
2. **A thin C++17 adapter** lives in `port/cpp/`. It exists only to bridge
   the template-heavy upstream interface — `graphics_device<HWDevice>`
   specializations, PSCs, shader classes — to the C core. All C++/template
   machinery is confined to `port/cpp/`; no template code in `port/c/`.
3. **ABI**: every `port/c/` entry point has `extern "C"` linkage. No C++
   types, no exceptions, no RTTI, no templates cross the boundary. Data
   crossing the boundary is plain structs/enums/pointers with documented
   ownership and lifetime.
4. **Ownership/lifetime/error contract** at the boundary (each C API):
   - who allocates and who frees every buffer (caller-provided arena
     discipline; no steady-state heap in the port);
   - validity windows for every pointer (e.g. DMA in-flight buffers must not
     be reused or freed — `CODING_CONVENTION.md` §18/§27);
   - every fallible call returns a status value that the caller must check
     (`CODING_CONVENTION.md` §5.4).
5. **No IDF headers in `src/egl_*.h`.** Platform knowledge enters only
   through the `HWDevice` contract (`src/egl_device.h`). The frozen snapshot
   gains no platform-specific includes.
6. **Precedence note**: for code style and safety rules,
   `CODING_CONVENTION.md` outranks this ADR and every other document for
   repo-owned code — its §1 rule hierarchy is authoritative for anything
   this repository owns. Upstream C++ (`src/`) is third-party and exempt
   from the convention; `port/` code is not.

## Alternatives considered

An all-C port (converting the upstream core to C) was rejected because it
would rewrite the frozen snapshot wholesale; an all-C++ platform layer under
MISRA C++ was rejected because the project convention pins MISRA C:2023 as
the safety baseline for repo-owned code.

## Consequences

- MISRA-analyzable surface: only `port/c/` needs MISRA conformance; the
  adapter is C++ and held to the convention's C++-applicable rules.
- The `extern "C"` boundary freezes a C ABI early; changing it later is a
  breaking change and needs an ADR.
- Two-language maintenance cost is accepted in exchange for the safety
  baseline on all code this repository owns.
- Phase 1 must confirm this ADR (or amend it) before creating `port/`, and
  must pick the build-config decision interacting with it (`EGL_DEBUG` vs
  `EGL_RETAIL`, see `docs/phase0/06`).

## Evidence

- MEASURED: upstream compiles as-is with the target toolchain —
  `-fsyntax-only` with the canonical command
  `riscv32-esp-elf-g++ -std=gnu++26 -march=rv32imafdc_zicsr_zifencei_zaamo_zalrsc -mabi=ilp32 -Isrc -fsyntax-only <file>`
  on `egl_core.cpp`, `egl_mesh.cpp`, `egl_texture.cpp`, `egl_rasterizer.cpp`,
  `egl_rasterizer_tiling.cpp`, `egl_rasterizer_cache.cpp`, and
  `-std=gnu++11|17|20|26` for a TU including all public headers (same
  `-march`/`-mabi` throughout; GCC 15.2.0; log:
  `/tmp/opencode/embergl-p4/syntax-check.log`, 2026-11-26).
- MEASURED: `src/egl_core.h:35` hard-defines `EGL_DEBUG`; the driver `.cpp`
  bodies are entirely inside `#ifdef ARDUINO`
  (`src/drivers/egl_device_ili9341.cpp:8`), so under IDF they compile to
  empty translation units.
- MEASURED: `CODING_CONVENTION.md` §1–§3 (C11, MISRA C:2023 primary
  baseline; third-party exemption in §1 and §36).
- UNVERIFIED `[U]`: whether the full MISRA required-rule set can be met by
  the platform port without exceptions for ISR/DMA-required constructs —
  to be demonstrated by Phase 1 code and deviation records.
