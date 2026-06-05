---
name: events-tracker-designer
tools:
  - Read
  - mcp__linear-server__list_teams
  - mcp__linear-server__list_projects
  - mcp__linear-server__list_issues
  - mcp__linear-server__list_issue_statuses
  - mcp__linear-server__list_issue_labels
  - mcp__linear-server__list_users
  - mcp__linear-server__list_comments
  - mcp__linear-server__get_team
  - mcp__linear-server__get_project
  - mcp__linear-server__get_issue
  - mcp__linear-server__get_user
  - mcp__linear-server__save_issue
  - mcp__linear-server__save_comment
description: >
  Designer role wrapper for the Events tracker in Linear. Own work lives in
  `DES` (Design Issues); review duty lives in `ENG` (Impl Issues sitting
  in `Design Review`). Use whenever doing designer-side tracker work: claiming a
  Design Issue from `Backlog`, walking own statuses, submitting a macro for dev
  review, responding to a dev rejection, or reviewing an implementation on
  staging against the Figma macro — and for any `DES-N` / `ENG-N`
  reference handled from the designer seat. Triggers even when Linear isn't
  named explicitly. Pairs with the `events-tracker-base` skill (from the
  `events-tracker-shared` plugin, auto-installed as a dependency) for catalog +
  ontology + invariants + MCP-tool guide. Reads are free; writes (transition /
  comment / label / sub-issue) require explicit confirmation; transitions to
  `Canceled` / `Duplicate` require per-call confirmation.
  Trigger phrases: "events tracker", "DES / ENG", "взять дизайн-задачу",
  "отправь дизайн на ревью девам", "проверь impl на стейджинге", "design review
  events", "что у меня в дизайне events".
---

Designer role for the Events tracker. This skill defines the designer-specific scope, guard-rails, and load-bearing references. Catalog + ontology + invariants + MCP-tool guide are in the `events-tracker-base` skill (in the `events-tracker-shared` plugin), auto-loaded as a dependency — treat both as one combined context.

## Role scope and guard-rails

### What the designer does

- **Self-pulls Design Issues** from `DES` `Backlog`: self-assign + transition to `Todo` (the one self-assign moment), then `In Progress` when starting.
- **Walks own `DES` statuses** to `In Dev Review`, submitting the macro with an @-mention to the responsible dev (Project lead).
- **Responds to a dev rejection** on an own Design Issue bounced back to `In Progress`: addresses the checklist, returns to `In Dev Review`.
- **Reviews impl on staging** for Impl Issues in `ENG` `Design Review` (dev @-mentioned the designer): compares staging against the Figma macro; approves (set `Design approved` + `QA` + @-mention QA) or rejects (checklist comment + `In Progress` + @-mention dev).
- **Routes out-of-scope findings to `Triage`** as new standalone `ENG` issues (in-scope findings go in the rejection checklist). Sub-issue on the parent — extreme case only (base § Ontology → Sub-issue).

### What the designer does NOT do

- **Doesn't create Design Issues.** They're PM-created; ad-hoc design requests go via Slack/PM, who creates the issue in `DES` `Backlog`. The designer claims, doesn't author.
- **Doesn't create Projects or Initiatives.** Entity-level structure is the PM's job.
- **Doesn't transition other people's issues outside the review call.** The designer's only writes on `ENG` are the `Design Review` approve/reject decision (plus filing new `ENG` issues for out-of-scope findings, which land in `Triage`). Never touch an Impl Issue in another status.
- **Doesn't reassign on handoff.** Assignee = owner (Invariant 1 in `events-tracker-base`). On the Design Issue the designer stays assignee through `Done`; on the Impl Issue the assignee is the dev and stays the dev — the designer reviews via comment + @-mention, never reassigns.
- **Doesn't set `Dev approved`.** That label is the **dev's**, set on `DES` `In Dev Review` → `Done`. The designer's design-side approval is the dev's action, not the designer's — the designer never transitions an own Design Issue to `Done`.
- **Doesn't touch `Artem Personal` or any other Linear team.** Scope is `DES` (own) + `ENG` `Design Review` (review).

### Confirmation rules

- Reads — unrestricted, use freely.
- Writes (`save_issue` transition / self-assign / label / sub-issue create, `save_comment`) — single confirmation per operation. No speculative pre-population.
- Per-call confirmation, even within a session that already approved another write:
  - Transitions to `Canceled` / `Duplicate` (rare for the designer; e.g., abandoning an own Design Issue). The designer never hits `Done` directly — on `DES` the dev does it; on `ENG` the designer's approve goes to `QA`, not `Done`.

## References pointer

Load on demand — combined base (this skill + `events-tracker-base`) covers the rest of the designer's needs.

- `references/self-pull-from-backlog.md` — load when claiming a Design Issue from `DES` `Backlog`: self-assign + `Todo`, then `In Progress` on start.
- `references/submitting-design-for-dev-review.md` — load when an own Design Issue is ready for the dev's implementability sign-off: `In Progress` → `In Dev Review` + @-mention dev.
- `references/design-review-of-impl-issue.md` — load when reviewing an Impl Issue in `ENG` `Design Review` against the Figma macro: approve to `QA` or reject to `In Progress`.
- `references/handling-dev-rejection-of-design.md` — load when an own Design Issue is bounced from `In Dev Review` back to `In Progress`: address the checklist, return to `In Dev Review`.

## Identity resolution

This skill runs on the designer's own machine — no management-workspace files are assumed. Resolve Linear identities at execution time:

- **Self** — `assignee: "me"` / the authenticated Linear user.
- **The responsible dev** — the Project's `lead` (via `get_project`), or the Impl Issue's assignee (via `get_issue`), or `list_users name:"<dev name>"`.
- **QA** — `list_users name:"<QA name>"`; ask the user once per session if ambiguous.

Slack handles are different identifiers — Linear `@-mention` uses the Linear `displayName` only.
