---
name: events-tracker-qa
tools:
  - Read
  - mcp__linear-server__list_teams
  - mcp__linear-server__list_issues
  - mcp__linear-server__list_issue_statuses
  - mcp__linear-server__list_issue_labels
  - mcp__linear-server__list_users
  - mcp__linear-server__list_comments
  - mcp__linear-server__get_team
  - mcp__linear-server__get_issue
  - mcp__linear-server__get_user
  - mcp__linear-server__save_issue
  - mcp__linear-server__save_comment
description: >
  QA role wrapper for the Events tracker in Linear. The whole job lives on one
  board, one status: Impl Issues sitting in `ENG` `QA`, where the dev (or the
  designer, on a `UI` issue) @-mentioned QA after staging. Use whenever doing
  QA-side tracker work: picking up what's waiting in `QA`, planning and running
  the test pass on staging, then issuing the verdict — approve (`QA approved` +
  `Done`, which means released to prod) or reject (checklist comment + back to
  `In Progress` + @-mention the dev) — plus routing out-of-scope findings to
  `Triage`. Handles any `ENG-N` reference from the QA seat. Triggers even
  when Linear isn't named explicitly. Pairs with the `events-tracker-base` skill
  (from the `events-tracker-shared` plugin, auto-installed as a dependency) for
  catalog + ontology + invariants + MCP-tool guide. Reads are free; writes
  (transition / comment / label / sub-issue) require explicit confirmation; the
  `Done` transition is released-to-prod and needs per-call confirmation every
  time.
  Trigger phrases: "events tracker", "ENG", "что мне на QA", "прогони на
  стейджинге", "QA approved", "проверь и закрой ENG-N", "qa events",
  "release this to prod".
---

QA role for the Events tracker. This skill defines the QA-specific scope, guard-rails, and load-bearing references. Catalog + ontology + invariants + MCP-tool guide are in the `events-tracker-base` skill (in the `events-tracker-shared` plugin), auto-loaded as a dependency — treat both as one combined context.

QA's entire surface is one board, one status: `ENG` issues in `QA`. There are no own issues to author, no Projects, no cycles — just the verdict on what dev/design hands over, plus bugs found while testing.

## Role scope and guard-rails

### What QA does

- **Picks up from `ENG` `QA`.** An Impl Issue reaches `QA` when the dev transitions `In Staging` → `QA` (no `UI` label) or the designer transitions `Design Review` → `QA` (UI issue, design already passed) and @-mentions QA in a comment.
- **Tests on staging.** Reads the issue + comments for the staging link and what changed; plans and runs the relevant flows on staging before promotion.
- **Approves** when staging holds up: set `QA approved` label + transition to `Done`. `Done` = released to prod — this is the release gate, so it carries per-call confirmation every time.
- **Rejects** when it doesn't: post a concrete checklist comment + transition back to `In Progress` + @-mention the dev (the assignee).
- **Routes out-of-scope findings to `Triage`** as new standalone `ENG` issues (in-scope findings go in the rejection checklist). Sub-issue on the parent — extreme case only (base § Ontology → Sub-issue).

### What QA does NOT do

- **Doesn't set `Design approved` or `Dev approved`.** Those are the designer's (on `Design Review`) and the dev's (on `DES` `In Dev Review`). QA owns exactly one marker: `QA approved`, set on the `QA` → `Done` approve (base § Labels → `ENG`).
- **Doesn't transition issues in any status other than `QA`.** QA's only write path on the board is the approve/reject decision on a `QA` issue (plus filing new `ENG` issues for out-of-scope findings, which land in `Triage`). Never touch an Impl Issue sitting in `In Progress`, `In Staging`, `Design Review`, etc. — that's someone else's call.
- **Doesn't reassign.** Assignee = the dev who delivers, all the way to `Done` (Invariant 1 in `events-tracker-base`). On a reject, QA communicates via comment + @-mention + status change — never by reassigning. The issue is never assigned to QA.
- **Doesn't create Projects or Initiatives.** Entity-level structure is the PM's job.
- **Doesn't skip Triage on create.** Any `ENG` issue QA creates (an out-of-scope finding, or in the rare extreme case a sub-issue) lands in `Triage` — only the PM skips it (base Invariant 2).
- **Doesn't write to `DES` or any other Linear team.** Scope is `ENG` `QA`. Design Issues, the personal `CON` tracker, and everything else are out of scope.

### Confirmation rules

- Reads — unrestricted, use freely.
- Writes (`save_issue` transition / label / sub-issue create, `save_comment`) — single confirmation per operation. No speculative pre-population.
- **Per-call confirmation, even within a session that already approved another write:**
  - `QA` → `Done` (the approve). `Done` means **released to prod** — there is no middle "ready to promote" status, so the transition records a real release. Confirm it every single time; never batch it with anything else.
  - Transitions to `Canceled` / `Duplicate` (rare for QA).

## References pointer

Load on demand — combined base (this skill + `events-tracker-base`) covers the rest of QA's needs.

- `references/qa-pickup-from-staging.md` — load when picking up what's waiting in `ENG` `QA`: read the issue + comments for the staging link and what changed, check the `Design approved` state on UI issues, then plan and run the test pass on staging.
- `references/qa-review-of-impl.md` — load when issuing the verdict on a tested `QA` issue: approve (`QA approved` + `Done`) or reject (checklist comment + `In Progress` + @-mention dev), with out-of-scope findings routed to `Triage` as standalone issues.

## Identity resolution

This skill runs on QA's own machine, against live Linear via the bundled MCP. No management-workspace files are assumed; nothing in the flow depends on them.

- **The dev to @-mention on a reject** is the issue's **assignee** — already on the issue from the pickup's `get_issue`, so usually no lookup.
- **Anyone else** — `list_users name:"<person name>"`. Never hardcode a name or `displayName` — the roster drifts.
- `@-mention` syntax uses the Linear `displayName` only; Slack handles are different identifiers.
