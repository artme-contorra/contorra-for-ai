# Creating a Project pair (Design + Impl)

Recipe for the PM's most choreographed flow: a Project spanning design + impl, with a Design Issue in `EVTDES` and an Impl Issue in `EVTENG`, linked by `blocks`. Loaded on demand from `events-tracker-pm` → `SKILL.md` → § References pointer. Entry-point trigger: "new feature for Events that needs design + impl", "create a project with design and impl tickets", "поставь дизайн + impl для X".

Out of scope: out-of-Project ad-hoc work (use Triage sweep instead — see `sweeping-triage.md`), Project edits, archive.

All steps run in chat. Single confirmation in step 6 covers the batch (Project + Design Issue + Impl Issue + relation).

## Procedure

### 1. Capture intent

Pull the feature name and the scope of work from conversation context. Confirm in chat:

- **Title** for the Project (Title Case, scannable; not a sentence). Examples: `Tournament leaderboard live updates`, `Inline AI assistant — round 1`.
- **Titles for the two Issues** — each names its own deliverable, readable without the Project context (see § Issue titles below). Draft both here; confirm with the rest of the batch.
- **One-paragraph description** for the Project: the problem, the scope, what "done" looks like. This is the shared context for both Issues (per the brevity rule — see `SKILL.md` → § Ontology — condensed → Brevity rule).
- **Responsible dev** — one of the `EVTENG` devs. They will be Project lead and Impl Issue assignee.
- **Initiative** — `Stability`, `AI Platform`, `UX`, or none. Most cross-role features attach to one. If unclear, ask: "под какую initiative — `Stability` / `AI Platform` / `UX` / без?".
- **Priority** — `1` Urgent, `2` High, `3` Medium (default), `4` Low.

If any of these are unclear, ask in one turn (batch the missing fields). Do not invent defaults beyond the priority default.

### 2. Resolve Linear `displayName` for dev + designer

Resolve the responsible dev's Linear `displayName` — either from `knowledge.md` § Team (if recent), or via `list_users name:"<dev name>"` (most current). Save it for use in `lead` on the Project and `assignee` on the Impl Issue.

Resolve the designer's `displayName` the same way for use in `assignee` on the Design Issue.

The skill intentionally doesn't hardcode handles — they drift when the roster or Linear identities change.

### Issue titles — verb-led imperatives, the verb carries the role

Every Issue title is an **imperative naming its own deliverable**, readable standalone on the team's board without opening the Project. The first word makes the role visible:

- **Design Issue** — starts with a design verb (`Design`, `Prototype`, `Explore`) + the actual design scope: which flows / screens / states.
- **Impl Issue** — starts with a build verb (`Add`, `Build`, `Support`, `Fix`, `Migrate`) + the shippable behaviour: what the user (or the system) gets.

Anti-patterns: identical or near-identical titles across the pair; mechanical ` — design` / ` — impl` suffixes or `Design /` tags; titles that only make sense inside the Project view.

Examples:

- Project `Customizable team names`: Design `Design team rename flow & settings UI` · Impl `Add custom team names to event settings`
- Project `Additional formats`: Design `Design scorecard layouts for best-2/3-of-4` · Impl `Support best 2-ball / best 3 of 4 / high-low scoring`

If the two titles come out near-identical, the split hasn't been thought through yet — ask what the design actually covers vs what the impl actually ships.

### 3. Draft the Design Issue (`EVTDES`)

- `team: "Events Design"`.
- `title` — per § Issue titles above: the design deliverable, standalone.
- `description` — minimal or skipped (context lives in Project description). If a one-line pointer helps the designer, add a single sentence like `Design for the [Project title] feature; see Project description.`.
- `state: "Backlog"`. PM creates skip Triage; `EVTDES` has no Triage anyway.
- `assignee: "<designer displayName>"` — resolved in step 2.
- `priority` — same as the Project unless there's a reason to diverge.
- `project: "<Project name>"` — attach after Project is created (see step 6 ordering).
- No labels at create (`Dev approved` is set later by the dev on approval).

### 4. Draft the Impl Issue (`EVTENG`)

- `team: "Events Engineering"`.
- `title` — per § Issue titles above: the shippable behaviour, standalone.
- `description` — minimal or skipped (same reason).
- `state: "Backlog"` (PM creates skip Triage). If pre-committed for the active cycle, use `state: "Todo"` + `cycle: "<n>"` — but Backlog is the safer default; let cycle planning pull it.
- `assignee: "<dev displayName>"` — resolved in step 2.
- `priority` — same as the Project unless there's a reason to diverge.
- `project: "<Project name>"`.
- `labels`:
  - exactly one of `"bug"` / `"feature"` / `"chore"` / `"tech-debt"` from the `Type` label group. Default for a new Project pair: `"feature"`. Ask if borderline (e.g., chore vs tech-debt).
  - `"UI"` if the work changes the user interface and needs a `Design Review` pass. Default for a cross-team Project: yes, set `"UI"` — the existence of a Design Issue is the strong signal that there's a UI surface. Ask only when the feature is partially backend (e.g., API plumbing that has a small UI affordance — does the `UI` flag belong?).
- `blocks` / `blockedBy` — leave empty at draft; set the cross-issue relation in step 6 after both Issues exist.

### 5. Show the full draft

Present the batch in chat:

- Project: name, description (one paragraph), lead, initiative, priority, teams (`Events Design` + `Events Engineering`).
- Design Issue: team, title, description, state, assignee, priority.
- Impl Issue: team, title, description, state, assignee, priority, project, labels.
- Cross-issue relation: Design Issue `blocks` Impl Issue.

Ask once for confirmation. Mention which fields you defaulted (e.g., "priority=Medium because nothing in context suggested otherwise").

### 6. Execute the batch

Ordering matters — Project must exist before Issues can attach to it, and both Issues must exist before the `blocks` relation can link them.

```
# 1. Create the Project
mcp__linear-server__save_project
  name: "<Project name>"
  description: "<one paragraph>"
  addTeams: ["Events Engineering", "Events Design"]
  lead: "<dev displayName>"           # resolved in step 2
  addInitiatives: ["Stability"]        # or "AI Platform" / "UX" / omit
  priority: 3                          # 0–4, defaults to Medium

# 2. Create the Design Issue (EVTDES)
mcp__linear-server__save_issue
  team: "Events Design"
  title: "<title>"
  description: ""                      # or single pointer sentence
  state: "Backlog"
  assignee: "<designer displayName>"   # resolved in step 2
  priority: 3
  project: "<Project name>"

# 3. Create the Impl Issue (EVTENG)
mcp__linear-server__save_issue
  team: "Events Engineering"
  title: "<title>"
  description: ""
  state: "Backlog"
  assignee: "<dev displayName>"        # resolved in step 2
  priority: 3
  project: "<Project name>"
  labels: ["feature", "UI"]

# 4. Set the blocks relation: Design → Impl
mcp__linear-server__save_issue
  id: "<EVTDES-N from step 2>"
  blocks: ["<EVTENG-N from step 3>"]
```

`blocks` is append-only (see `SKILL.md` → § MCP-tool guide → `save_issue` — update). Setting it once on the Design Issue creates the relation; Linear renders the inverse `blockedBy` on the Impl Issue automatically.

### 7. Echo result in chat

- Project URL + `EVTDES-N` + `EVTENG-N`.
- Note that the Impl Issue stays blocked until the Design Issue reaches `Done` (when the designer's macro gets `Dev approved`).

## Notes

- **Why `lead` = dev, not designer.** Per `2026-06-03_events-delivery-flow-design.md` § 3 and `2026-06-04_events-tracker-ontology-design.md` § 2: the dev owns delivery from design phase to ship. Designer is participant on the Project (via the Design Issue) but not lead.
- **Cross-team project membership.** Linear's Project entity supports multiple teams via `addTeams`. Both teams see the Project in their own Project view; Issues remain in their respective team boards.
- **When to skip the Design Issue.** If the feature is genuinely no-UI (e.g., pure backend / CI / infra), create the Project on `Events Engineering` only (`addTeams: ["Events Engineering"]`) and skip steps 3 + the `blocks` relation. The Impl Issue then carries no `UI` label.
- **When to skip the Impl Issue at create time.** Rare — usually only if the design lead time is long enough that creating the Impl Issue now would clutter the dev's queue. Document the decision in the Project description and create the Impl Issue at cycle planning instead.
- **Cycle attachment.** Don't pre-attach the Impl Issue to a cycle here unless the user explicitly pre-committed it. Cycle planning pulls from `Backlog` (see `cycle-planning.md`).
