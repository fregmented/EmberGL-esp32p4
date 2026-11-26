---
adr-id: 0002
title: Repository root as an ESP-IDF component (Phase 1 proposal)
status: proposed
date: 2026-11-26
deciders: Phase 0 repository initialization
---

# 0002 — Repository root as an ESP-IDF component (Phase 1 proposal)

## Context

The upstream repository is an Arduino library layout (`library.properties`,
`src/`, `examples/`) with no ESP-IDF packaging. The Phase 0 audit found:

- No `CMakeLists.txt`, `idf_component.yml`, `Kconfig`, or `sdkconfig` at the
  repo root; `src/` contains no CMake registration.
- `example/` is a standalone IDF project that builds for `esp32p4` but does
  not reference EmberGL sources at all (Phase 0 build verification only).
- ESP-IDF supports two consumption models for the library: (a) make the
  repository root an IDF component, or (b) keep the library unpackaged and
  list `src/egl_*.cpp` directly in the consuming project's `main` component.

Phase 0 does not implement packaging; this ADR records the proposed shape so
Phase 1 starts from a reviewed design.

## Decision (proposed — not implemented in Phase 0)

1. In **Phase 1**, make the repository root an ESP-IDF component named
   `embergl`:
   - A root `CMakeLists.txt` registering the upstream C++ sources
     (`src/*.cpp`, `src/drivers/*.cpp`) with include dir `src/`.
   - A root `idf_component.yml` manifest declaring the component, with
     `targets: [esp32p4]`.
2. The component includes the **root** `src/` — the frozen snapshot is not
   copied or moved. Consumers compile the frozen sources through the
   component; the frozen-paths rule (ADR-0001) is unaffected.
3. The upstream Arduino layout (`library.properties`, Arduino sketch
   examples under `examples/`) stays untouched; both packaging models
   coexist in one repository.
4. The stock example consumes the component via a relative path manifest
   dependency:

   ```yaml
   dependencies:
     embergl: { path: ../.. }
   ```

   (Plus the existing managed display/touch dependencies.)
5. The component declares its target as `esp32p4`. Wider target support is
   out of scope for this ADR.

**Explicitly: none of these files exist in Phase 0.** There is no root
`CMakeLists.txt` and no root `idf_component.yml` today; creating them is a
Phase 1 change gated on owner approval of this proposal.

## Consequences

- Consumers get `idf.py add-dependency`-style consumption of a pinned
  snapshot without submodules or vendoring.
- The example remains buildable in isolation (its `/tmp`-copy build passes
  without the component in Phase 0).
- Phase 1 must resolve before implementing: which IDF components the
  component `REQUIRES`/`PRIV_REQUIRES` (candidates: none for the pure core;
  platform drivers belong to `port/`, not the core); whether the component
  name in the registry namespace can be secured; how the Arduino `examples/`
  are excluded from IDF globs.
- Building the component for non-P4 targets is untested; the manifest should
  not advertise targets that are not verified.

## Evidence

- MEASURED: repository root contains no `CMakeLists.txt`, `idf_component.yml`,
  `Kconfig`, or `sdkconfig` (directory listing, 2026-11-26).
- MEASURED: `example/` builds for `esp32p4` standalone
  (`/tmp/opencode/embergl-p4/example-build`, build exit 0, 0 warning lines);
  its `main/CMakeLists.txt` registers only `hello_world_main.c`.
- MEASURED: `example/main/idf_component.yml` pins `idf: ==6.1.1`,
  `espressif/esp_lcd_st7701: ==2.0.2`, `espressif/esp_lcd_touch_gt911: ==1.1.3`,
  `targets: [esp32p4]`.
- UNVERIFIED `[U]`: registry availability of an `embergl` component name and
  the exact `REQUIRES` closure for Phase 1.
