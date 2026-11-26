# Issue tracker: GitHub

Issues and specs for this repo live as GitHub issues on
`fregmented/EmberGL-esp32p4`. Use the `gh` CLI for all operations.

## Conventions

- **Always pin the repository**: every command carries
  `-R fregmented/EmberGL-esp32p4`. This repository is a **fork** of
  `EmberGL-org/EmberGL`; without the flag, `gh` can resolve to the wrong
  repository, and inside a fork `gh` may default to the upstream parent.
- **Issues-enabled prerequisite**: issue operations fail while Issues are
  disabled. Phase 0 enabled them (`gh repo edit fregmented/EmberGL-esp32p4
  --enable-issues`); if a command reports "has disabled issues", re-check
  with `gh repo view fregmented/EmberGL-esp32p4 --json hasIssuesEnabled`
  before doing anything else.
- **File issues only into the user's fork.** Never file into
  `EmberGL-org/EmberGL` (upstream) or `mattpocock/skills` (where the skills
  originate) unless the user owns that repository. Before the first
  `gh issue create` of a session, confirm the target:
  `gh repo view fregmented/EmberGL-esp32p4 --json nameWithOwner`.
- **Create an issue**: `gh issue create -R fregmented/EmberGL-esp32p4 --title "..." --body "..."`.
  Use a heredoc for multi-line bodies.
- **Read an issue**: `gh issue view <number> -R fregmented/EmberGL-esp32p4 --json number,title,body,labels,comments`.
- **List issues**: `gh issue list -R fregmented/EmberGL-esp32p4 --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'`
  with appropriate `--label` and `--state` filters.
- **Make an issue a sub-issue of a parent**:
  `gh issue create -R fregmented/EmberGL-esp32p4 --parent <parent> ...`, or
  `gh issue edit <parent> -R fregmented/EmberGL-esp32p4 --add-sub-issue <child>`
  afterwards (`gh` 2.94+). Without sub-issues, put `Part of #<parent>` at the
  top of the child body.
- **Blocking**: GitHub's native issue dependencies. Add an edge with
  `gh issue create -R fregmented/EmberGL-esp32p4 --blocked-by <n>,<n> ...` or
  `gh issue edit <child> -R fregmented/EmberGL-esp32p4 --add-blocked-by <blocker>`
  (`gh` 2.94+). Without dependencies, fall back to a
  `Blocked by: #<n>, #<n>` line at the top of the child body. A ticket is
  unblocked when every blocker is closed.
- **Count open blockers**:
  `gh api repos/fregmented/EmberGL-esp32p4/issues/<n> --jq .issue_dependencies_summary.blocked_by`
  (open blockers only; 0 means startable).
  `gh issue view <n> -R fregmented/EmberGL-esp32p4 --json blockedBy` also
  lists closed blockers, so filter:
  `--jq '[.blockedBy.nodes[] | select(.state == "OPEN")] | length'`.
- **Comment on an issue**: `gh issue comment <number> -R fregmented/EmberGL-esp32p4 --body "..."`.
- **Apply / remove labels**:
  `gh issue edit <number> -R fregmented/EmberGL-esp32p4 --add-label "..."` /
  `--remove-label "..."`. The label vocabulary is in
  [triage-labels.md](triage-labels.md).
- **Close**: post the explanation with
  `gh issue comment <number> -R fregmented/EmberGL-esp32p4 --body "..."`,
  then `gh issue close <number> -R fregmented/EmberGL-esp32p4`. On an
  already-closed issue, `gh issue close --comment` drops the comment and
  still exits 0.

GitHub shares one number space across issues and PRs, so a bare `#42` may be
either: resolve with `gh pr view 42 -R fregmented/EmberGL-esp32p4` and fall
back to `gh issue view 42 -R fregmented/EmberGL-esp32p4`.

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` only if this repo decides to
treat external PRs as feature requests; `/triage` reads this flag.)_

Pull requests in this repository are integration artifacts under the git
workflow (`docs/agents/general_instruction/git_workflow.md`): they target
`develop`, and agents open but do not merge them. They do not flow through
the triage labels.

## When a skill says "publish to the issue tracker"

Create a GitHub issue in `fregmented/EmberGL-esp32p4`.

## When a skill says "fetch the relevant ticket"

Read it as in **Read an issue** above.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single issue with **child** issues as
tickets.

- **Map**: a single issue labelled `wayfinder:map`, holding the Notes /
  Decisions-so-far / Fog body.
  `gh issue create -R fregmented/EmberGL-esp32p4 --label wayfinder:map`.
- **Child ticket**: an issue linked to the map as a GitHub sub-issue (see
  **Make an issue a sub-issue of a parent**). Where sub-issues aren't
  enabled, add the child to a task list in the map body and put
  `Part of #<map>` at the top of the child body. Labels: `wayfinder:<type>`
  (`research`/`prototype`/`grilling`/`task`). Once claimed, the ticket is
  assigned to the driving dev.
- **Blocking**: see **Blocking** under Conventions.
- **Frontier query**: list the map's open children (`gh issue list --state open`,
  scoped to the map's sub-issues / task list), drop any with an open blocker
  (see **Count open blockers**, or an open issue in the `Blocked by` line) or
  an assignee; first in map order wins.
- **Claim**: `gh issue edit <n> -R fregmented/EmberGL-esp32p4 --add-assignee @me`,
  the session's first write.
- **Resolve**: `gh issue comment <n> -R fregmented/EmberGL-esp32p4 --body "<answer>"`,
  then `gh issue close <n> -R fregmented/EmberGL-esp32p4`, then append a
  context pointer (gist + link) to the map's Decisions-so-far.

Note: the `wayfinder:*` labels do **not** exist yet in this repository (only
the five triage labels and GitHub defaults existed at Phase 0). Create them
with `gh label create` before the first wayfinding run, and record their
creation in the map issue.
