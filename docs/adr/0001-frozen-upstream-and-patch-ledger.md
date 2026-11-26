---
adr-id: 0001
title: Frozen upstream snapshot and patch ledger
status: accepted
date: 2026-11-26
deciders: Phase 0 repository initialization
---

# 0001 — Frozen upstream snapshot and patch ledger

## Context

This fork specializes EmberGL for the ESP32-P4. EmberGL is a third-party MIT
library; every avoidable divergence from upstream raises the cost of pulling
future upstream fixes and makes it impossible to tell which behavior is
original and which is fork-local. At the same time, a port will eventually
need small, deliberate deviations (e.g. a build-configuration hook). Phase 0
needs a verifiable baseline and a rule for how deviations may ever happen.

Verified baseline facts:

- Upstream release: tag `v0.1.1`, commit `cedba19604ddfa6f2de5679cf398952de3a554d`
  (local tag `v0.1.1` resolves to the same hash; upstream
  `github.com/EmberGL-Org/EmberGL` serves the same `refs/tags/v0.1.1`).
- License: MIT, © 2022 Jarkko Lempiainen (`LICENSE` line 3; every `src/`
  file carries the same header, e.g. `src/egl_core.h:4-5`).
- At Phase 0 start, `git diff --stat v0.1.1 -- src examples LICENSE library.properties`
  is empty: those paths are byte-identical to the tag. The only fork-local
  commits at that point are the removal of `.github/FUNDING.yml` and agent-doc
  additions (`6c19745` → `533dc7d`).

## Decision

1. **Freeze** `src/`, `examples/`, and `doc/` at the upstream baseline.
   No agent or human edits these paths without a patch-ledger entry (below)
   and owner approval.
2. Also freeze upstream metadata `LICENSE`, `VERSION`, `library.properties`;
   MIT redistribution requires preserving the notice.
3. **Patch ledger.** Any future deviation from the frozen snapshot requires
   exactly one ledger entry, recorded as a new ADR-0001 ledger section (or a
   linked tracker issue referenced from it), with this schema:

   | Field | Content |
   | --- | --- |
   | Patch ID | `P<seq>` (monotonic) |
   | Baseline hash | full SHA the patch applies to |
   | Path(s)/symbol(s) | files and symbols touched |
   | Change | what was altered, factually |
   | Rationale | why a fork-local change is unavoidable |
   | ADR/issue | linking ADR or tracker issue |
   | Verification | how correctness was established (commands, results) |
   | Upstream status | `not-reported` / `PR <url>` / `fixed-upstream <tag>` |

4. **Traceability diff gate** — must exit 0 on every PR that does not carry an
   approved ledger entry:

   ```bash
   git diff --exit-code v0.1.1 -- src examples doc
   git diff --exit-code v0.1.1 -- src examples LICENSE library.properties
   ```

5. Phase 0 creates **no** patches: the ledger starts empty.

## Consequences

- Upstream fixes can be merged with trivial conflict cost while the ledger is
  empty; every added entry makes future merges progressively harder, which is
  the intended pressure to keep deviations rare.
- Reviewers and agents have a mechanical check for accidental upstream edits.
- The gate is a guard, not a proof of intent: a passing gate says the
  snapshot is intact, not that the fork's own code is correct.
- If upstream advances past v0.1.1, adopting a new baseline is a new decision
  (new ledger reset with owner approval), not a silent re-freeze.

## Evidence

- MEASURED: `git rev-parse v0.1.1` → `cedba19604ddfa6f2de5679cf398952de3a554d`;
  `git tag -l` → `v0.1.0`, `v0.1.1`.
- MEASURED: `git diff --stat v0.1.1 HEAD` at Phase 0 start → only
  `.github/FUNDING.yml` (−1), `.gitmodules` (+3), submodule pointer (+1);
  no `src/`/`examples/`/`doc/` paths.
- MEASURED: `LICENSE:3` = `Copyright (c) 2022 Jarkko Lempiainen`;
  `VERSION` = `0.1.1`; `library.properties` `version=0.1.1`.
- MEASURED (network, 2026-11-26): `git ls-remote https://github.com/EmberGL-Org/EmberGL`
  returns `refs/tags/v0.1.1` = `cedba196…` — local tag matches upstream.
