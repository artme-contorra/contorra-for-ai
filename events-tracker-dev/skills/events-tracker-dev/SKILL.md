---
name: events-tracker-dev
version: "0.1"
tools:
  - Read
  - mcp__linear-server__list_teams
  - mcp__linear-server__list_projects
  - mcp__linear-server__list_issues
  - mcp__linear-server__list_issue_statuses
  - mcp__linear-server__list_issue_labels
  - mcp__linear-server__list_cycles
  - mcp__linear-server__list_users
  - mcp__linear-server__list_comments
  - mcp__linear-server__get_team
  - mcp__linear-server__get_project
  - mcp__linear-server__get_issue
  - mcp__linear-server__get_user
  - mcp__linear-server__save_issue
  - mcp__linear-server__save_comment
description: >
  Developer role wrapper for the Events tracker in Linear. Own work lives in
  `EVTENG` (Impl Issues, assignee = you); review duty lives in `EVTDES` (Design
  Issues sitting in `In Dev Review`). Use whenever doing dev-side tracker work:
  picking up an assigned Impl Issue from `Todo`, walking it `In Progress` →
  `In Review` → `In Staging`, handing off to `Design Review` or `QA` after
  staging, responding to a Design Review / QA bounce, or reviewing a designer's
  macro for implementability — and for any `EVTENG-N` / `EVTDES-N` reference
  handled from the dev seat. Triggers even when Linear isn't named explicitly.
  Pairs with the `events-tracker-base` skill (from the `events-tracker-shared`
  plugin, auto-installed as a dependency) for catalog + ontology + invariants +
  MCP-tool guide. Reads are free; writes (transition / comment / label /
  sub-issue) require explicit confirmation; transitions to `Done` (only on
  `EVTDES` `In Dev Review` → `Done`) / `Canceled` / `Duplicate` require per-call
  confirmation.
  Trigger phrases: "events tracker", "EVTENG / EVTDES", "взять задачу в работу",
  "выложил на стейджинг", "отдай на ревью / QA", "проверь макет на
  реализуемость", "вернули задачу, поправить", "what's assigned to me events",
  "review the design for implementability".
---

Developer role for the Events tracker. This skill defines the dev-specific scope, guard-rails, and load-bearing references. Catalog + ontology + invariants + MCP-tool guide are in the `events-tracker-base` skill (in the `events-tracker-shared` plugin), auto-loaded as a dependency — treat both as one combined context.

The consumer is one of the team's web devs (there are two — never assume which; resolve identity via `assignee: "me"` for own issues and `list_users` for others).

## Role scope and guard-rails

### What the dev does

- **Picks up an assigned Impl Issue** from `EVTENG` `Todo` (active cycle) into `In Progress`. Pre-flight: any blocking Design Issue is `Done`, and the Project context is understood.
- **Walks own `EVTENG` statuses** `In Progress` → `In Review` → `In Staging` (manual until the GitHub integration is turned on; see § Workspace integration). Assignee stays the dev throughout.
- **Hands off after `In Staging`**: `UI` label present → `Design Review` + @-mention the designer (with the staging link); no `UI` → `QA` + @-mention QA.
- **Responds to a bounce**: an own Impl Issue sent back to `In Progress` from `Design Review` or `QA` — parse the reviewer's checklist, fix, walk forward again, hand off skipping already-cleared phases.
- **Reviews a designer's macro** for implementability on `EVTDES` `In Dev Review` (the designer @-mentioned the dev): approve (`Dev approved` label + `Done`) or reject (checklist comment + `In Progress` + @-mention designer).
- **May dispute the `UI` label** on an own Impl Issue via comment + label removal (base Invariant 3) when the change doesn't actually touch the UI.
- **Owns the feature from design phase to ship** (delivery-flow § 3): the dev self-tests before every hand-off. The review phases *validate* the work; they are not the first-detection step. "QA will catch it" is not a fallback.

### What the dev does NOT do

- **Doesn't transition an own Impl Issue to `Done`.** `Done` on `EVTENG` is **QA's** call after the `QA` phase. The dev's hand-off ceiling is `Design Review` / `QA`.
- **Doesn't set `Design approved` (designer's) or `QA approved` (QA's).** The only marker the dev sets is `Dev approved`, and only on `EVTDES` `In Dev Review` → `Done`.
- **Doesn't make a reviewer's call.** `Design Review` → `QA` is the designer's; `QA` → `Done` is QA's. The dev moves work *into* a review state and addresses bounces; it doesn't approve on a reviewer's behalf.
- **Doesn't reassign on handoff.** Assignee = owner (Invariant 1 in `events-tracker-base`). On an own Impl Issue the dev stays assignee to `Done`; on a Design Issue under review the assignee is the designer and stays the designer — the dev reviews via comment + @-mention, never reassigns.
- **Doesn't create Projects or Initiatives.** Entity-level structure is the PM's job.
- **Doesn't skip Triage on create.** A dev-created `EVTENG` issue (e.g., a sub-issue bug) lands in `Triage` — only the PM skips it (base Invariant 2). @-mention the PM so it gets swept.
- **Doesn't touch `Artem Personal` or any other Linear team.** Scope is `EVTENG` (own) + `EVTDES` `In Dev Review` (review).

### Confirmation rules

- Reads — unrestricted, use freely.
- Writes (`save_issue` transition / label / sub-issue create, `save_comment`) — single confirmation per operation. No speculative pre-population.
- Per-call confirmation, even within a session that already approved another write:
  - `EVTDES` `In Dev Review` → `Done` (this is the dev's one `Done` transition, and it releases the `blocks` relation — see `dev-review-of-design.md`).
  - Transitions to `Canceled` / `Duplicate`.

## References pointer

Load on demand — combined base (this skill + `events-tracker-base`) covers the rest of the dev's needs.

- `references/picking-up-impl-issue.md` — load when starting an assigned Impl Issue: `EVTENG` `Todo` (active cycle) → `In Progress`, after pre-flight checks.
- `references/handoff-after-staging.md` — load when an own Impl Issue has reached `In Staging` and needs routing to a review phase: `Design Review` (if `UI`) or `QA`.
- `references/dev-review-of-design.md` — load when a Design Issue in `EVTDES` `In Dev Review` needs the dev's implementability sign-off: approve to `Done` or reject to `In Progress`.
- `references/responding-to-bounce.md` — load when an own Impl Issue was bounced back to `In Progress` from `Design Review` / `QA`: fix, walk forward, hand off skipping cleared phases.

## Workspace integration

This skill runs on the dev's own machine, not in a shared management workspace — there is no `knowledge.md` or `state.md` to lean on. Resolve identity directly from Linear:

- **Own issues** — filter with `assignee: "me"`; the dev running the skill is the assignee.
- **Other people** (designer, QA, the other dev) — resolve `displayName` via `list_users name:"<person name>"` at execution time, or read a Project's `lead` (`get_project`) for the responsible dev. Never hardcode a name or `displayName`; the roster drifts.
- **GitHub / PR conventions** — your team's branch model, PR self-test checklist, and UI-PR screenshot + designer @-mention convention live in your own repo / PR-conventions doc, not here. Tracker transitions are manual until the Linear↔GitHub integration is enabled; once it is, branch → `In Progress`, PR ready → `In Review`, merge → `In Staging` become automatic (base § Statuses → `EVTENG`).
- **`events-tracker-base`** — the only sibling skill this one depends on: catalog, ontology, invariants, MCP-tool guide. Don't restate it here.
