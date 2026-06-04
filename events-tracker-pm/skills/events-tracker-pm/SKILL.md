---
name: events-tracker-pm
version: "0.1"
tools:
  - Read
  - mcp__linear-server__list_teams
  - mcp__linear-server__list_projects
  - mcp__linear-server__list_issues
  - mcp__linear-server__list_issue_statuses
  - mcp__linear-server__list_issue_labels
  - mcp__linear-server__list_initiatives
  - mcp__linear-server__list_cycles
  - mcp__linear-server__list_users
  - mcp__linear-server__list_comments
  - mcp__linear-server__get_team
  - mcp__linear-server__get_project
  - mcp__linear-server__get_issue
  - mcp__linear-server__get_initiative
  - mcp__linear-server__get_user
  - mcp__linear-server__save_issue
  - mcp__linear-server__save_project
  - mcp__linear-server__save_comment
  - mcp__linear-server__save_initiative
  - mcp__linear-server__save_status_update
  - mcp__linear-server__create_issue_label
description: >
  PM / tracker-owner role wrapper for the Events tracker in Linear (teams
  `EVTENG` and `EVTDES`). Use whenever doing PM-side tracker work: creating a
  Project pair (Design + Impl), sweeping `EVTENG` `Triage`, cycle planning,
  status reviews, or any reference to an `EVTENG-N` / `EVTDES-N` identifier
  done from the PM seat. Triggers even when Linear isn't named explicitly.
  Pairs with the `events-tracker-base` skill (from the `events-tracker-shared`
  plugin, auto-installed as a dependency) for catalog + ontology + invariants
  + MCP-tool guide. Reads are free; writes (create / edit / transition /
  comment) require explicit confirmation; status transitions to `Done` /
  `Canceled` / `Duplicate`, Project archive, cycle rewrites, and bulk
  description rewrites require per-call confirmation.
  Trigger phrases: "events tracker", "EVTENG / EVTDES", "разгрести triage",
  "запланируй цикл events", "создай проект events", "design + impl",
  "что у девов/дизайнера в трекере events".
---

PM / tracker-owner role for the Events tracker. This skill defines the PM-specific scope, guard-rails, and load-bearing references. Catalog + ontology + invariants + MCP-tool guide are in the `events-tracker-base` skill (in the `events-tracker-shared` plugin), auto-loaded as a dependency — treat both as one combined context.

## Role scope and guard-rails

### What the PM does

- Creates **Projects** (with both teams attached when the work spans design + impl).
- Creates **Design Issue + Impl Issue pairs** inside Projects; sets `blocks` Design → Impl.
- Sweeps **`EVTENG` Triage** (priority, type label, project, assignee, transition).
- Plans **cycles** in `EVTENG` (pulls Impl Issues from `Backlog` to `Todo`, attaches to cycle).
- Status reviews, comments, ad-hoc edits where it serves the team.

### What the PM does NOT do

- **Doesn't write code, do design work, or do QA work.** The skill orchestrates the tracker; it doesn't deliver the work the tracker holds.
- **Doesn't transition issues across review boundaries on behalf of reviewers.** A `Design Review` → `QA` transition is the designer's call; a `QA` → `Done` is QA's. PM can comment, nudge in Slack, or reassign-the-owner if the actual owner changed — but doesn't approve on someone else's behalf.
- **Doesn't reassign on handoff.** Assignee = owner (Invariant 1 in `events-tracker-base`). PM only changes assignee when ownership genuinely moves (e.g., re-routing an Impl Issue from one dev to the other).

### Confirmation rules

- Reads — unrestricted, use freely.
- Writes (`save_issue`, `save_project`, `save_comment`, `save_initiative`, `save_status_update`, `create_issue_label`) — single confirmation per operation. No speculative pre-population.
- Per-call confirmation, even within a session that already approved another write:
  - Transitions to `Done` / `Canceled` / `Duplicate`.
  - Project archive (via `save_project` `state`).
  - Cycle rewrites (re-attaching multiple issues across cycles).
  - Bulk description rewrites.

## References pointer

Load on demand — combined base (this skill + `events-tracker-base`) covers the rest of the PM's needs.

- `references/creating-project-pair.md` — load when creating a cross-team Project with a Design + Impl pair (the most choreographed PM flow).
- `references/sweeping-triage.md` — load when working through `EVTENG` `Triage` entries: priority + label + project + assignee + transition.
- `references/cycle-planning.md` — load at weekly cycle planning on `EVTENG`: pulling Impl Issues from `Backlog` into `Todo` for the active cycle.

## Workspace integration

- `knowledge.md` — canon on team members (roles, working rhythm, traits, Linear `displayName`). Update there, not in this skill.
- `state.md` — current PM focus; may reference specific `EVTENG-N` / `EVTDES-N` items. After a sync, surface entries that need a status nudge.
- `linear-context` — **deprecated** for Events work; still owns Artem's personal `CON` tracker until that's migrated. Don't widen this skill to cover `CON`.
- `jira-context` — out of scope. Jira `EV` is being superseded by this Linear setup; for legacy `EV-N` references use the `jira-context` skill.
- `slack-context` — orthogonal. `@-mention` syntax in Linear uses `displayName`; Slack handles + UserIDs are different identifiers in `slack-context`.
- `github-context` — orthogonal. Branch / PR conventions described there; tracker side described here.
