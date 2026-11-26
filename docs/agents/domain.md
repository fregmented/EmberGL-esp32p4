# Domain Docs

How the engineering skills should consume this repo's domain documentation
when exploring the codebase.

## Layout: single-context

This repository is single-context: one `GLOSSARY.md` at the repo root and one
shared `docs/adr/` directory. The context is "EmberGL ESP32-P4 port" (see
the header of [GLOSSARY.md](../../GLOSSARY.md)).

There is no `GLOSSARY-MAP.md` and no per-context glossary directories. Do not
create them without a domain-modeling decision.

## Before exploring, read these

- **`GLOSSARY.md`** at the repo root.
- **`docs/adr/`**: read ADRs that touch the area you're about to work in.
  Current set: 0001 (frozen upstream + patch ledger), 0002 (root as IDF
  component, proposed), 0003 (language/safety boundary, proposed), 0004
  (capability-gated accelerator policies, proposed).

If any of these files don't exist, **proceed silently**. Don't flag their
absence; don't suggest creating them upfront. The `/domain-modeling` skill
(reached via `/grill-with-docs` and `/improve-codebase-architecture`) creates
them lazily when terms or decisions actually get resolved.

## Use the glossary's vocabulary

When your output names a domain concept (in an issue title, a refactor
proposal, a hypothesis, a test name), use the term as defined in
`GLOSSARY.md`. Don't drift to synonyms the glossary explicitly avoids.

Pay attention to the glossary's *(design)* markers: terms such as *platform
port*, *P4 backend*, *accelerator policy*, or *worker* name planned
structure that does not exist in Phase 0. Never write about them as if they
were implemented.

If the concept you need isn't in the glossary yet, that's a signal: either
you're inventing language the project doesn't use (reconsider) or there's a
real gap (note it for `/domain-modeling`).

## Flag ADR conflicts

If your output contradicts an existing ADR, surface it explicitly rather
than silently overriding:

> _Contradicts ADR-0002 (root as an ESP-IDF component), but worth reopening
> because…_

For this repository, the ADRs most likely to be contradicted by a hasty
proposal are 0001 (any edit to `src/`, `examples/`, `doc/` needs a ledger
entry) and 0003 (no C++ across the `extern "C"` boundary, no IDF headers in
`src/egl_*.h`).
