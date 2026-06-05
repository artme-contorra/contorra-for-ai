# Sweeping `ENG` Triage

Recipe for the daily PM flow: walk each `ENG` `Triage` entry, set the four missing facets (priority, type label, project, assignee), then transition out. Loaded on demand from `events-tracker-pm` → `SKILL.md` → § References pointer. Entry-point trigger: "разгрести triage", "что в triage у events", "sweep events triage".

Out of scope: `DES` (no `Triage` on the Design team — see `SKILL.md` → § Ontology — condensed → Statuses). Issues already past `Triage`. Bulk-import or migration sweeps.

All steps run in chat. One confirmation per entry (a sweep over N entries = N confirmations, batched per entry).

## Procedure

### 1. Pull the Triage queue

```
mcp__linear-server__list_issues
  team: "Events Engineering"
  state: "Triage"
  orderBy: "createdAt"
  limit: 50
```

Show count and a one-line summary per entry: `ENG-N — title — creator — createdAt — has-description?`. If the queue is empty, say so and stop.

If creator info is needed (to gauge context), fetch with `get_issue` on individual entries — `list_issues` returns the basics.

### 2. Per-entry sweep

For each entry, walk these four decisions in order. Default to **enriching the entry's description with new context** when conversation surfaces extra detail — don't let facts surfaced during the sweep drop on the floor.

#### 2a. Priority

Linear scale: `1` Urgent / `2` High / `3` Medium / `4` Low. Default `3` Medium. Extremes are for actual edge cases (incident, time-critical, deferred polish).

State your pick + one-line rationale in chat ("urgent — blocks tomorrow's release", "low — polish on settings screen").

#### 2b. Type label

Exactly one of `bug` / `feature` / `chore` / `tech-debt` (bare label name — see `SKILL.md` → § Ontology — condensed → Labels). Linear enforces mutual exclusion via the `Type` label group.

- `bug` — something broken vs. expected behaviour.
- `feature` — new user-visible capability.
- `chore` — routine work, no user-visible change (config, deps, infra hygiene).
- `tech-debt` — internal refactor that improves engineering quality without changing behaviour.

Borderline (chore vs tech-debt, bug vs feature for an edge case) — state pick + reasoning; user can override in step 3.

#### 2c. `UI` label (separate question from type)

Set `UI` if and only if the work changes the user interface and needs a `Design Review` pass downstream (see `SKILL.md` → § Invariants → `UI` semantics). PM is the one who sets it at creation/triage time; the dev can dispute later via comment + label removal.

For Triage-incoming items the creator (often a teammate or support) usually didn't think about this — explicit PM call here.

#### 2d. Project

- `mcp__linear-server__list_projects team:"Events Engineering"` if needed to refresh the snapshot in `SKILL.md` → § Map → Projects.
- If the issue fits an existing Project, pass `project: "<name>"`.
- If not, leave unattached — out-of-Project work is a normal default for small / one-off items (see `SKILL.md` → § Ontology — condensed → Entity gradient).
- If the issue is big enough to be its own Project but isn't yet, flag in chat: "this looks Project-sized — promote, or sweep as standalone Issue?". Don't silently create a Project mid-sweep.

#### 2e. Assignee

- Pick one of the `ENG` devs — the one who owns this. Match the Project lead if attached to a Project; otherwise pick by area / workload (see `knowledge.md`).
- Resolve the assignee's Linear `displayName` from `knowledge.md` § Team or via `list_users name:"<dev name>"`. Pass as `displayName`, name, email, or UUID — **not** `assigneeId`.
- If unclear which dev, ask in one turn: "which dev owns this?".

#### 2f. Description sanity check

The entry came from a non-PM and may be thin. If the conversation surfaces extra context (cause, reproduction steps, related issue, link), append a single `## Update YYYY-MM-DD` section to the description and confirm in the same batch. Don't strip the original author's text — append, don't overwrite. The goal: new context surfaced during sweep lands somewhere, not on the floor.

#### 2g. Language

Tracker language is English (base Invariant 5). If the entry's title or description came in Russian, translate both as part of the sweep — title always; description translated rather than appended (this is the one case where rewriting the author's text is correct; keep the meaning, note `*(translated from Russian at triage)*` at the bottom if the original phrasing might matter).

### 3. Show draft + confirm + transition

For each entry, present the batch:

- `ENG-N` — title.
- Decisions: priority, labels (`bug|feature|chore|tech-debt` + optional `UI`), project (or "standalone"), assignee.
- Description addition if any.
- Target state: `Backlog` (default) or `Todo` + `cycle: "<n>"` (only if explicitly pre-committed to the active cycle — usually for an urgent item the team agreed to pull in immediately).

Single confirmation per entry, then:

```
mcp__linear-server__save_issue
  id: "ENG-N"
  priority: 3
  labels: ["feature", "UI"]
  project: "<Project name>"           # omit if standalone
  assignee: "<dev displayName>"        # resolved per step 2e
  state: "Backlog"                     # or "Todo"
  cycle: "<n>"                         # only when state: "Todo"
  description: "<original + ## Update YYYY-MM-DD>"   # only when enriched
```

`labels` on `save_issue` replaces the label set on the issue. If the entry came in with a pre-set label (rare for Triage), include it in the array to keep it. Marker labels (`Design approved`, `QA approved`) are not set at Triage — they appear later in the flow.

`blockedBy` / `blocks` / `relatedTo` are append-only — pass them on this call only if a dependency surfaced during sweep (see `SKILL.md` → § MCP-tool guide → `save_issue` — update).

### 4. Move to the next entry

Print `ENG-N` → new state in chat. Continue with the next Triage entry until the queue is empty or the user asks to pause.

### 5. End-of-sweep summary

When the queue empties (or the user pauses), summarize: count swept, count promoted to `Todo` + cycle, count remaining. Surface any entry that was deferred for an explicit reason (e.g., "needs the designer's input on UI question — left in Triage, post in Slack").

## Notes

- **PM-created issues skip Triage.** This sweep is only for entries created by non-PM (team-members, support, automation). When the PM creates an issue directly, they already set priority + label + project + assignee at create time, so it lands in `Backlog` (or `Todo`). See `SKILL.md` → § Invariants → Triage routing.
- **`DES` has no Triage.** Ad-hoc design requests come via Slack and the PM creates the Issue directly in `DES` `Backlog` — no equivalent sweep needed there.
- **Cycle attachment is the exception.** Default end-state is `Backlog`, not `Todo` + cycle. Cycle planning (see `cycle-planning.md`) is when most Impl Issues move from `Backlog` to `Todo`. Use `Todo` + cycle in Triage sweep only for items the team explicitly agreed to pull in mid-cycle.
- **One confirmation per entry, not per sweep.** Each entry's decisions are different — batching the whole queue into one approval loses the per-item review. The cost of N confirmations on a sweep is acceptable; the alternative is silently shipping a wrong assignee or label.
- **Don't transition to `Done` / `Canceled` / `Duplicate` from Triage.** Those are post-work outcomes. If the Triage entry is genuinely a duplicate of an existing issue, use `state: "Duplicate"` + `duplicateOf: "<existing-N>"` — but treat this as a destructive transition (see `SKILL.md` → § Role scope and guard-rails → Confirmation rules: separate per-call confirmation).
