# Triage Labels

The skills speak in terms of five canonical triage roles. This file maps
those roles to the actual label strings used in this repo's issue tracker
(`fregmented/EmberGL-esp32p4`).

| Label in mattpocock/skills | Label in our tracker | Meaning | State in this repo |
| -------------------------- | -------------------- | ------- | ------------------ |
| `needs-triage` | `needs-triage` | Maintainer needs to evaluate this issue | Created in Phase 0 (`FBCA04`) |
| `needs-info` | `needs-info` | Waiting on reporter for more information | Created in Phase 0 (`D876E3`) |
| `ready-for-agent` | `ready-for-agent` | Fully specified, ready for an AFK agent | Created in Phase 0 (`0E8A16`) |
| `ready-for-human` | `ready-for-human` | Requires human implementation | Created in Phase 0 (`1D76DB`) |
| `wontfix` | `wontfix` | Will not be actioned | Pre-existing GitHub default (`ffffff`) — do not recreate |

Do not create duplicates. If a `gh label create` reports the label already
exists, keep the existing label and verify its color/description with
`gh label list -R fregmented/EmberGL-esp32p4 --json name,color,description`
instead of forcing a rewrite.

When a skill mentions a role (e.g. "apply the AFK-ready triage label"), use
the corresponding label string from this table.

Issues are filed per [issue-tracker.md](issue-tracker.md): always pinned to
`-R fregmented/EmberGL-esp32p4`, never into the upstream parent repository.
