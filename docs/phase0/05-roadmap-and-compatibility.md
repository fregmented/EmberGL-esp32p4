# Phase 0 — 05: Prioritized Roadmap and Compatibility Requirements

Covers **deliverable 9** (prioritized roadmap) and **deliverable 10**
(compatibility requirements). Phase goals are fixed project decisions; the
content under each goal summarizes the design constraints recorded in the
other phase0 docs. Exit criteria are verifiable statements; "hardware run"
means an actual flash-and-run with human approval.

## 1. Roadmap

### Phase 0 — Project Initialization (this phase)
Design-only baseline: fact base, governance docs, ADRs, build verification of
the stock example, tracker setup. No platform code.
**Exit criteria:** docs complete and committed; protected-path gates pass;
`example/` builds for `esp32p4` in a /tmp copy; Issues + labels enabled;
PR open against `develop`.

### Phase 1 — Baseline Port
Goal: get the unmodified EmberGL core rendering through a first device
adapter on ESP32-P4 — software-only present, correctness equal to the host
golden reference.
Includes: confirm/implement ADR-0002 (root component) and ADR-0003
(language boundary); decide the `EGL_DEBUG`/`EGL_RETAIL` build config
(`06` R15); first `port/c` + `port/cpp` code; one scene of the suite
(`04` S2) matching golden.
Depends on: Phase 0. Blocked if the frozen-snapshot ledger gains entries
(ADR-0001 review first).

### Phase 2 — Memory and Execution Architecture
Goal: make memory and execution explicit — arena contract, buffer carving,
cache-maintenance policy, present path choice, async completion model.
Depends on: Phase 1 (running baseline to measure against).

### Phase 3 — Hardware Acceleration
Goal: capability-gated accelerator policies (ADR-0004) implemented with
software twins — PPA present-stage ops, DMA2D 2-D transfers; `off`/`auto`/
`force` semantics; tolerance policy enforced against twins.
Depends on: Phase 2 (buffer contracts must exist first).

### Phase 4 — Processor-Specific Optimization
Goal: single-core performance work under the measurement discipline
(`04` §5): compiler/codegen, cache-friendliness, tile-order interplay;
comparisons only via the standing matrix.
Depends on: Phase 3 (fallback-complete paths so optimizations can't break
correctness).

### Phase 5 — Multicore Optimization
Goal: workers ≤ cores−1, disjoint tiles, per-worker buffers, locked shared
vcache, deterministic mode; static tile-shader wrapper hazard resolved
beforehand (`06` R16).
Depends on: Phase 4; requires the determinism harness from `04`.

### Phase 6 — General-Purpose Rendering Optimization
Goal: library-level (not SoC-specific) wins that upstream also benefits
from — feed upstream where appropriate under ADR-0001's ledger rules
(prefer upstream PRs over forks patches).
Depends on: Phases 4–5 insights.

### Phase 7 — Release Readiness
Goal: versioned component publication, docs, CI on the build gate, license
audit, registry availability decision.
Depends on: all prior phases; requires ADR-0002 accepted and implemented.

Dependency chain: 0 → 1 → 2 → 3 → 4 → 5 → 6; 7 depends on 1–6. Phases 4/5/6
may overlap after their predecessors' exit criteria hold, but not before.

## 2. Compatibility requirements

| Dimension | Requirement | Status |
| --- | --- | --- |
| ESP-IDF | Target **6.1.x**, validated on **6.1.1**. Other 6.1.x minors expected compatible but only 6.1.1 is verified; other majors unsupported until validated | 6.1.1 VERIFIED; 6.1.x `[U]` |
| Compiler | GCC 15.2.0 (riscv32-esp-elf) is the primary, validated toolchain; Clang for RISC-V is **best-effort** only if it appears and is explicitly validated — no claim today | GCC VERIFIED; Clang `[U]` |
| Silicon revisions | P4 rev ≥ v3 required for 360→400 MHz default deltas and 250 MHz PSRAM; revision-sensitive behavior must be detected at runtime, never assumed | `[U]` — no silicon in hand |
| EmberGL API/ABI | Upstream public API unchanged; the port adds a device layer. The `extern "C"` boundary of ADR-0003 is the port's ABI; breaking it requires a new ADR | Phase 0 direction |
| Arduino upstream | `library.properties`, `examples/`, Arduino workflows unaffected by the fork (ADR-0002) | Protected paths VERIFIED |
| License | MIT © 2022 Jarkko Lempiainen preserved verbatim; fork additions carry compatible notices; attribution in README | VERIFIED (`LICENSE:3`) |
| Versioning / registry | Fork versioning scheme (e.g. `0.1.1-p4.x`) and component-registry publication are **decisions to make in Phase 7**; nothing published yet | `[U]` |
| Build reproducibility | Builds only from /tmp copies with the pinned activation script; dependencies locked (`dependencies.lock` tracked) | VERIFIED for the stock example |

## 3. Phase 0 exit criteria checklist

- [x] Governance docs (AGENTS.md, GLOSSARY.md, ADRs, agent docs) committed.
- [x] `docs/phase0/` 00–06 committed, mapping all 13 deliverables.
- [x] Protected-path gates pass (`git diff --exit-code v0.1.1 …`).
- [x] Stock example build-verified for `esp32p4` outside the repo
      (target_rc=0, build_rc=0, 0 warning lines — see `00`).
- [x] GitHub Issues enabled + five triage labels present.
- [x] No `src/` edits, no performance claims, no hardware claims.
