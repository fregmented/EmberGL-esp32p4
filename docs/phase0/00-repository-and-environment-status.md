# Phase 0 — 00: Repository and Environment Status

Covers **deliverable 1** (repository and environment status), **deliverable
12** (created/modified files), and **deliverable 13** (build verification).
Everything marked MEASURED was executed on 2026-11-26 during Phase 0.

## 1. Git baseline and preflight

| Check | Result |
| --- | --- |
| Working tree at start | Clean (`git status --porcelain=v1 -b` → `## develop...origin/develop`, zero entries) |
| Branch at start | `develop` |
| HEAD at start | `533dc7d` (`533dc7dfc53f6a0d6cf17f8b61bccd7db1d5f61f`) |
| `origin/develop` at start | `533dc7d` (identical; `git merge --ff-only origin/develop` → "Already up to date") |
| Work branch | `chore/phase0-project-initialization` created from `develop` @ `533dc7d` (did not previously exist, locally or on remote) |
| Fork lineage | `6c19745` (upstream `main` state) → `c5ac273` (agent-docs submodule) → `533dc7d` (`Update initial commits`: example scaffold + `.gitignore` + CODING_CONVENTION.md) |

History note: the pre-Phase-0 fact analysis ran against `c5ac273` with staged
files; `533dc7d` (made outside this analysis) committed that state. The
pre-existing root `.gitignore` is a generic OS/CMake/Python template; Phase 0
extended it with the ESP-IDF artifact section below rather than replacing it.

## 2. Protected-path verification

```
$ git diff --exit-code v0.1.1 -- src examples doc
$ git diff --exit-code v0.1.1 -- src examples LICENSE library.properties
```

Both exited 0 (no output) at preflight and again after all Phase 0 changes —
MEASURED. `src/`, `examples/`, `doc/`, `LICENSE`, `library.properties`,
`VERSION`, and the `docs/agents/general_instruction` submodule were not
touched. The `.gitmodules`-tracked submodule pointer is unchanged.

## 3. Environment

| Item | Value (MEASURED) |
| --- | --- |
| ESP-IDF | v6.1.1 (`~/.espressif/v6.1.1/esp-idf`) |
| Activation | `source /home/hanwool/.espressif/tools/activate_idf_v6.1.1.sh` |
| Compiler | `riscv32-esp-elf-gcc 15.2.0` (crosstool-NG `esp-15.2.0_20251204`) |
| Python | 3.12.3 |
| `gh` | 2.104.0, authenticated as the fork owner (scopes withheld) |

## 4. Build verification (deliverable 13)

Executed **outside the repository**, per AGENTS.md:

```bash
cd <repository root>
REPO="$PWD"
WORK=/tmp/opencode/embergl-p4
rm -rf "$WORK/example-build-src" "$WORK/example-build"
cp -a "$REPO/example" "$WORK/example-build-src"
source /home/hanwool/.espressif/tools/activate_idf_v6.1.1.sh
cd "$WORK/example-build-src"
idf.py set-target esp32p4            # → example-set-target.log
idf.py -B "$WORK/example-build" build  # → example-build.log
grep -Eic 'warning:' example-build.log
```

| Check | Result |
| --- | --- |
| `idf.py set-target esp32p4` exit code | **0** (target_rc=0) |
| `idf.py build` exit code | **0** (build_rc=0) |
| Build result | **PASS** (`EmberGLExample.bin` 0x28f10 bytes; smallest app partition 0x100000, 84% free) |
| Build log lines matching `warning:` | **0** |
| Logs | `/tmp/opencode/embergl-p4/example-set-target.log`, `/tmp/opencode/embergl-p4/example-build.log` |
| Build tree | `/tmp/opencode/embergl-p4/example-build-src` (source copy), `/tmp/opencode/embergl-p4/example-build` (build dir) |
| Repository status after build | clean — no artifacts leaked into the repo |

Configure-stage facts captured from the same run (MEASURED):

- Generated `sdkconfig` defaults: `CONFIG_IDF_TARGET="esp32p4"`,
  `CONFIG_ESPTOOLPY_FLASHSIZE="2MB"`, `# CONFIG_SPIRAM is not set`,
  `CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ=400`, `CONFIG_CACHE_L2_CACHE_SIZE=0x20000`,
  `CONFIG_CACHE_L2_CACHE_LINE_SIZE=64`, `CONFIG_CACHE_L1_CACHE_LINE_SIZE=64`,
  `CONFIG_ESP_MAIN_TASK_STACK_SIZE=3584`.
- Resolved components include `esp_lcd`, `esp_driver_dma`, `esp_mm`;
  **`esp_driver_ppa` is absent** from the component list.
- Managed components resolved: `espressif/esp_lcd_st7701 2.0.2`,
  `espressif/esp_lcd_touch_gt911 1.1.3`, `espressif/esp_lcd_touch 1.2.1`,
  `espressif/cmake_utilities 0.5.3` (recorded in `dependencies.lock`).

**Explicit non-claim:** the stock example is a stock IDF `hello_world`
scaffold plus display/touch component pins. It does **not** compile, link,
or exercise any EmberGL code. This PASS verifies the toolchain/target
environment only — not the library, not any port, and not any hardware.

## 5. GitHub tracker setup result

| Operation | Result |
| --- | --- |
| `gh repo view fregmented/EmberGL-esp32p4` | fork of `EmberGL-org/EmberGL`; `hasIssuesEnabled` was `false` |
| `gh repo edit fregmented/EmberGL-esp32p4 --enable-issues` | OK; `hasIssuesEnabled` now `true` |
| `gh label create needs-triage` (FBCA04) | OK |
| `gh label create needs-info` (D876E3) | OK |
| `gh label create ready-for-agent` (0E8A16) | OK |
| `gh label create ready-for-human` (1D76DB) | OK |
| `wontfix` | pre-existing GitHub default — left untouched, not recreated |
| Post-verification | all five labels present via `gh label list --json name,color,description`; Issues enabled confirmed |

No tokens were printed; issue creation was not exercised (no issues were
filed in Phase 0).

## 6. Created and modified files (deliverable 12)

| Path | Change | Commit |
| --- | --- | --- |
| `AGENTS.md` | new — agent operating instructions | 1 |
| `GLOSSARY.md` | new — canonical vocabulary | 1 |
| `.gitignore` | modified — appended ESP-IDF artifact section to the pre-existing template | 1 |
| `docs/adr/0001-frozen-upstream-and-patch-ledger.md` | new | 2 |
| `docs/adr/0002-root-as-esp-idf-component.md` | new | 2 |
| `docs/adr/0003-language-and-safety-boundary.md` | new | 2 |
| `docs/adr/0004-capability-gated-accelerator-policies.md` | new | 2 |
| `docs/agents/issue-tracker.md` | new | 3 |
| `docs/agents/triage-labels.md` | new | 3 |
| `docs/agents/domain.md` | new | 3 |
| `docs/phase0/00…06` (7 files) | new | 3 |
| `README.md` | modified — prefix prepended; upstream suffix byte-identical | 4 |

Nothing under `src/`, `examples/`, `doc/`, `example/` was modified; no
`port/`, root `CMakeLists.txt`, `idf_component.yml`, `CLAUDE.md`, or
`GLOSSARY-MAP.md` was created.

## 7. Deliverable map (13 deliverables → docs)

| Doc | Deliverables covered |
| --- | --- |
| `00-repository-and-environment-status.md` | 1 repository/environment status · 12 created/modified files · 13 build verification |
| `01-upstream-audit-and-feature-inventory.md` | 2 upstream audit · 4 rendering feature inventory |
| `02-hardware-capability-matrix.md` | 3 verified hardware capability matrix |
| `03-platform-integration-architecture.md` | 5 platform integration architecture · 7 memory/execution proposal |
| `04-candidate-optimizations-and-benchmark-methodology.md` | 6 candidate optimizations · 8 benchmark methodology |
| `05-roadmap-and-compatibility.md` | 9 prioritized roadmap · 10 compatibility requirements |
| `06-risks-assumptions-open-questions.md` | 11 risks, assumptions, open questions |

## 8. Limitations

- No flash, no serial monitor, no on-target run of any kind (explicit scope
  exclusion; requires human approval).
- No performance benchmark exists; `04` defines the methodology only, and
  its comparison matrix is empty by design.
- The stock example build does not validate EmberGL (see §4 non-claim).
- Panel/touch controller support is unverified; the example's managed
  dependencies are scaffold only.
- Offline component-manager resolution was not proven; the Phase 0 build
  resolved from the local component cache and/or the registry
  (`dependencies.lock` recorded).
