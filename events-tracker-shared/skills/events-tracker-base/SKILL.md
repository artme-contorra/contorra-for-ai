---
name: events-tracker-base
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
description: >
  Base catalog + ontology + invariants + Linear MCP-tool guide for the Events
  tracker (teams `EVTENG` Events Engineering and `EVTDES` Events Design) at
  Contorra. Auto-loaded as a dependency of any `events-tracker-*` role plugin
  (PM, dev, designer, QA) — pair this skill with a role plugin for full
  coverage. Use whenever any Events tracker work comes up in this workspace,
  even when Linear isn't named explicitly. Strictly scoped to the two Events
  teams; `Artem Personal` and any other Linear teams are out of scope (use
  `linear-context` for `CON` until migrated).
  Trigger phrases: "events tracker", "EVTENG / EVTDES", "EVTENG-N", "EVTDES-N",
  "трекер events", "Stability / AI Platform / UX initiative", "Events Project",
  "что в трекере events".
---

Base for the Events tracker. Not a workflow — a space map: catalog + ontology + invariants + MCP guide. Pair with a role plugin (`events-tracker-pm`, `-dev`, `-designer`, `-qa`) for role-specific scope, guard-rails, and load-bearing references.

## Map

### Team

| Role | Count | Notes |
|---|---|---|
| PM / tracker-owner | 1 | Artem. Skips `Triage` on create. |
| Dev (web) | 2 | Linear assignee for Impl Issues in `EVTENG` and `lead` on Projects. |
| Designer | 1 | Linear assignee for Design Issues in `EVTDES`. |
| QA | 1 | Reviewer on `EVTENG` `QA` status; not assigned (assignee = dev — see § Invariants). |

People canon (current roster, names, working rhythm): `knowledge.md` → § Team in the management workspace. Slack handles + UserIDs: `slack-context` skill.

Linear `displayName` for each person is intentionally not hardcoded here — it drifts when the roster or handles change. At execution time resolve from `knowledge.md` or via `list_users name:"<person name>"` (most current). Mobile team is out of scope.

### Linear teams

| Key | Name | ID | Purpose |
|---|---|---|---|
| `EVTENG` | Events Engineering | `23ad419d-cc6f-423d-a7f7-18cccb2833cc` | Impl Issues (web). Weekly cycles. |
| `EVTDES` | Events Design | `3088ba1c-456d-4357-af55-ec4b288480bc` | Design Issues. No cycles. |

Issue identifiers: `EVTENG-N`, `EVTDES-N`. URL pattern: `https://linear.app/contorra/issue/<KEY>-<N>/<slug>`. Workspace name in URLs is `contorra`.

### Initiatives (strategic directions)

| Name | ID | One-line direction |
|---|---|---|
| `Stability` | `03a75eda-b4db-493f-8c3d-3dc70d588837` | Stop high-impact failures from reaching organizers. Monitoring, CI, tests, staged rollout. |
| `AI Platform` | `277b7df7-116d-4df7-8570-e0db1791826d` | Make Events drivable from outside its UI. One API for web + mobile + LLMs. |
| `UX` | `c0090932-e01e-4ba6-8ac4-f0d7d43d510f` | Reduce friction in the organizer's path. Session replay + qualitative loop. |

`Growth / Marketing` initiative (`3e481c2e-...`) is pending deletion — UI-only delete, the Linear MCP has no `delete_initiative`. Don't attach new projects to it.

### Projects — snapshot (not load-bearing)

Snapshot as of 2026-06-04; refresh via `list_projects team:"Events Engineering"` (and same for `Events Design`):

- **Cross-team** (both `EVTENG` + `EVTDES`): `[Events] Test` (sandbox), `Automated testing` (Stability), `Additional formats`.
- **`EVTENG` only:** `Customizable team names`, `Security` (Stability), `CI & release pipeline` (Stability), `Error monitoring` (Stability), `Session replay` (UX), `Backend refactor` (AI Platform).
- All current projects are in `Backlog` status except `[Events] Test` (Planned). Most have no lead set yet.

## Ontology — condensed

Canonical source (in the management workspace): `research/processes/2026-06-04_events-tracker-ontology-design.md`. This section is the operating summary; canonical wins on conflict.

### Entity gradient (Initiative → Project → Issue → Sub-issue)

- **Initiative** — strategic banner, cross-quarter. Opt-in for projects. No owner. Three live: `Stability`, `AI Platform`, `UX`.
- **Project** — time-bounded deliverable. **Cross-team container** — attach `EVTENG` + `EVTDES` when the work spans design + impl. Threshold from Issue: work spans weeks and naturally splits across roles. Lead = the responsible dev (= future Impl Issue assignee). Description required (one paragraph max).
- **Issue** — single assignee, discrete unit.
  - *Inside Project:* role-split into Design Issue (`EVTDES`, assignee = designer) + Impl Issue (`EVTENG`, assignee = dev). Description minimal or skipped — context lives in Project description.
  - *Outside Project:* ad-hoc (bug / chore / polish). **Description required** — no Project context to lean on. Default for small / one-off work.
- **Sub-issue** — extreme cases only; default: not used. Three valid uses: (a) rare in-scope review gap that must be its own tracked item while still blocking the parent (parent stays in review until sub-issue → `Done`) — **not** the default review outcome, which is checklist comment + bounce (in-scope) or a new standalone `Triage` issue (out-of-scope); (b) explicit blocker ("waiting on X to do Y"; owner = the person being waited on); (c) rare size-split.

### Statuses

**`EVTENG`** — 11-status set:

| Name | Type | Meaning |
|---|---|---|
| `Triage` | triage | Incoming from non-PM. Sweep needed (§ Invariants). |
| `Backlog` | backlog | Accepted; not scheduled into a cycle. |
| `Todo` | unstarted | Committed in current cycle; not started. |
| `In Progress` | started | Branch / draft PR open, dev working. |
| `In Review` | started | PR open, awaiting code review. |
| `In Staging` | started | Merged; deployed to staging. Awaiting designer / QA. |
| `Design Review` | started | Designer reviewing impl on staging. |
| `QA` | started | QA testing impl on staging. |
| `Done` | completed | Released to prod. |
| `Canceled` | canceled | Won't ship. |
| `Duplicate` | duplicate | Same as another issue. |

GitHub integration (branch → `In Progress`, ready → `In Review`, merge → `In Staging`) is deferred — transitions are manual until turned on.

**`EVTDES`** — 7-status set (no `Triage`, no PR / staging / QA gates):

| Name | Type | Meaning |
|---|---|---|
| `Backlog` | backlog | Accepted; designer hasn't picked up. |
| `Todo` | unstarted | Designer self-assigned, ready to start. |
| `In Progress` | started | Designer working on macro. |
| `In Dev Review` | started | Macro ready; dev signing off on implementability. |
| `Done` | completed | Dev approved; `blocks` releases the dependent Impl Issue. |
| `Canceled` | canceled | Won't ship. |
| `Duplicate` | duplicate | Same as another issue. |

### Labels

**`EVTENG`:**

| Label | Casing | Set by | When |
|---|---|---|---|
| `bug` / `feature` / `chore` / `tech-debt` | lowercase | PM — at create (own issues) or at Triage sweep (incoming) | Mandatory exactly-one. Linear **label group** `Type` enforces mutual exclusion natively. |
| `UI` | all caps | PM at create | Impl Issue changes the UI and needs a `Design Review` pass. Assignee can dispute via comment + label removal. |
| `Design approved` | Title case | Designer | Set on `Design Review` → `QA` approve. Survives bounces — a later QA reject doesn't undo it. |
| `QA approved` | Title case | QA | Set on `QA` → `Done` approve. Survives bounces. |

Area labels are **not pre-created**. Add organically if filtering pressure shows up.

**`EVTDES`:**

| Label | Casing | Set by | When |
|---|---|---|---|
| `Dev approved` | Title case | Dev | Set on `In Dev Review` → `Done` approve. Marks design as implementation-ready. |

No Type group on `EVTDES`. No `UI` (design issues are by definition UI).

### Priority

Linear native 0–4: `0` None / `1` Urgent / `2` High / `3` Medium / `4` Low. Real field, **not** encoded in backlog order.

- **Project priority** — coarse, intuition-based dashboard signal.
- **Issue priority** — default `Medium`. Extremes (`Urgent`, `Low`) for actual edge cases.

### Blocking relations

- **Design Issue → `blocks` → Impl Issue.** PM sets at create time when making a Project pair. Design must reach `Done` for the relation to release Impl from being blocked.
- **Sub-issue bugs from review** (extreme case — see § Entity gradient) implicitly block their parent (parent stays in review status until all sub-issues close).
- `blockedBy` / `blocks` on `save_issue` are **append-only** — to remove a relation use `removeBlockedBy` / `removeBlocks`. See § MCP-tool guide.

### Saved views (human entry points)

The team's curated Linear views (per ontology canon § 7): **Triage**, **My active** (assignee = me, active statuses), **Current cycle**, **By Project**, **In flight** (`In Review` / `In Staging` / `Design Review` / `QA`). These are human board entry points — agents reconstruct the same slices with `list_issues` filters and don't depend on the views existing.

### Brevity rule (descriptions)

One paragraph at most on Project and on out-of-Project Issues. **In-Project Issues:** description minimal or skipped — context lives in the Project description. Detail belongs in linked design docs, PR descriptions, or comments, not in the canonical entity body. Never edit a description to reflect status (status lives in state + comments).

## Invariants

These hold across the entire tracker. Violations break downstream conventions.

1. **No reassignment on review handoff.** Assignee = owner (the actor who carries the issue to `Done`), never the reviewer. Design Issue assignee = designer all the way; Impl Issue assignee = dev all the way. Reviewers communicate via comment + @-mention; status transitions are made by whoever's call it is (owner moves work into a review state; reviewer approves / rejects). Rationale: "what am I on the hook for" dashboards must answer *who delivers*, not *who looked at it most recently*.
2. **Triage routing is `EVTENG` only.** All non-PM-created `EVTENG` issues land in `Triage`. PM sweep (daily or as items arrive) sets priority, type label, project, assignee, then transitions to `Backlog` (or `Todo` + cycle if pre-committed). PM-created issues skip `Triage`. `EVTDES` has no `Triage` — ad-hoc design requests come via Slack and the PM creates the issue directly in `Backlog`.
3. **`UI` label semantics.** PM sets `UI` at creation when the Impl Issue changes the user interface and needs a `Design Review` pass. The assignee (dev) can dispute via comment + label removal; PM should not silently re-set it. Lives outside the `Type` group.
4. **Marker labels persist across bounces.** `Design approved`, `QA approved`, `Dev approved` survive when an issue bounces back to `In Progress` from a later review. The next `In Staging` skips the already-cleared phase unless the relevant surface actually changed again. Re-entry doesn't redo passed gates.
5. **Tracker language is English.** All tracker content — titles, descriptions, comments — is written in English, regardless of the conversation language around it. The tracker is the English-language canonical record; Slack and meetings can be Russian. Items that arrive in Russian (e.g., Triage submissions) get translated when processed.

## MCP-tool guide

### Always scope by team

Pass `team: "Events Engineering"` or `team: "Events Design"` (or the team UUID — see § Map) on every list / get tool that accepts it. Without it, queries pull workspace-wide, including `Artem Personal` and any other private teams.

### `list_issues`

- `state` accepts type or name: by type `state: "started"` covers `In Progress` + `In Review` + `In Staging` + `Design Review` + `QA` (5 statuses) on `EVTENG`; by name use the exact status string (`"Design Review"`, `"In Staging"`).
- `assignee` — pass the `displayName`, name, email, or UUID (resolve via § Map → Team or `list_users`). `"me"` also valid.
- `priority`: `1` Urgent, `2` High, `3` Medium, `4` Low.
- `cycle` — by name, number, or ID. For "current cycle on EVTENG" first fetch via `list_cycles teamId:<UUID> type:"current"` (see gotcha below).
- `label` — bare label name (`"bug"`, `"UI"`, `"Design approved"`).
- `project` — by name, ID, or slug.
- `query` — searches title + description.

### `save_issue` — create

Required for create: `team`, `title`. Common pattern:

- `team: "Events Engineering"` or `team: "Events Design"`.
- `title`, `description` (one paragraph for in-Project; fuller for out-of-Project).
- `state: "Backlog"` (PM-created skips Triage). For an immediate cycle commit, use `state: "Todo"` + `cycle: "<n>"`.
- `priority: 3` (Medium default).
- `assignee: "<displayName>"` — resolve from § Map → Team or `list_users`. **Not** `assigneeId`.
- `project: "<name>"` when inside a Project.
- `labels: ["bug"]` (or `feature` / `chore` / `tech-debt`) on `EVTENG`. Bare label name, no `Type/` prefix even though Linear renders the group.
- Add `"UI"` to the labels array when the Impl Issue changes the UI.
- `blocks: ["EVTENG-12"]` / `blockedBy: ["EVTDES-7"]` for cross-team dependency.

### `save_issue` — update

- Pass `id` (e.g. `"EVTENG-3"`). Omitting `id` creates a new issue.
- **Status transitions go through `state`** — `state: "Done"` / `"In Progress"` / `"Design Review"` etc. Accepts state type (`"completed"`), name, or ID.
- `labels` is **replace-semantics** — passing `labels` overwrites the issue's whole label set. Re-send the existing labels plus your addition (e.g. existing + `"Design approved"`), or you'll silently wipe the `Type` label / `UI` / earlier markers.
- `blockedBy` / `blocks` / `relatedTo` are **append-only** (contrast `labels` above) — passing them again does **not** replace the set. Use `removeBlockedBy` / `removeBlocks` / `removeRelatedTo` to remove. This avoids silently wiping someone else's added relation.

### `save_project`

- Create: `name` + at least one of `addTeams` / `setTeams`. For cross-team Projects: `addTeams: ["Events Engineering", "Events Design"]`.
- `lead: "<displayName>"` — the responsible dev (= future Impl Issue assignee).
- `addInitiatives: ["Stability"]` to attach to an initiative. Projects can also live without an initiative.
- `description` — one paragraph (brevity rule).
- `priority: 0–4` — coarse, intuition-based.

### `save_comment`

- `issueId: "EVTENG-12"` + `body` for a top-level thread.
- `parentId: "<comment-id>"` for a reply.
- `@-mention` syntax: `@displayName` (e.g. `@<dev displayName>`). Resolve `displayName` per § Map → Team or `list_users`.

### `list_cycles` gotcha

`list_cycles` requires `teamId` (UUID), **not** the team key or name. Pass the UUID from the Map section:

- `EVTENG`: `23ad419d-cc6f-423d-a7f7-18cccb2833cc`
- `EVTDES`: `3088ba1c-456d-4357-af55-ec4b288480bc` (cycles not used on `EVTDES`; expected to return `[]`).

`type: "current"` returns the active cycle; `"next"` and `"previous"` exist too. Omit `type` for the whole list.

### Initiative deletion gotcha

There is **no `delete_initiative`** MCP tool. Initiative deletion is UI-only in Linear. `Growth / Marketing` is pending UI-only delete; don't try to delete via MCP.

### String encoding

Per the linear-server MCP server instruction: pass content with **literal newlines and special characters**, not escape sequences (e.g. write real newlines in a markdown body, not `\n`). Applies to `description` and `body`.

## Canonical-source pointer

- Ontology canon (management workspace): `research/processes/2026-06-04_events-tracker-ontology-design.md`.
- Delivery-flow canon (substrate, discipline, why the tracker shape is what it is): `research/processes/2026-06-03_events-delivery-flow-design.md`.

If this skill conflicts with the canonical docs, **canonical wins** — patch the skill, not the doc. Flag the discrepancy in chat when surfaced.
