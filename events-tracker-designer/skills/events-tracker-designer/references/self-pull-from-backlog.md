# Self-pulling a Design Issue from `DES` Backlog

Recipe for the designer's entry flow: claim a PM-created Design Issue sitting in `DES` `Backlog`, self-assign and transition to `Todo`, then to `In Progress` when work actually starts. Loaded on demand from `events-tracker-designer` → `SKILL.md` → § References pointer. Entry-point trigger: "взять дизайн-задачу", "что мне взять в дизайне", "claim a design issue", "pull from DES backlog".

Out of scope: creating Design Issues (PM-created — see `SKILL.md` → § Role scope → What the designer does NOT do). `ENG` review work (see `design-review-of-impl-issue.md`). Cycle pulling — `DES` runs **no cycles**; the designer self-organizes from `Backlog` at own cadence (base § Multi-team structure).

All steps run in chat. One confirmation per transition (self-assign + `Todo` is one write; `In Progress` is a separate write made when work starts).

## Procedure

### 1. Pull the `DES` Backlog

```
mcp__linear-server__list_issues
  team: "Events Design"
  state: "Backlog"
  orderBy: "updatedAt"
  limit: 50
```

Show count and a one-line summary per entry: `DES-N — title — priority — project — blocks?`. If the queue is empty, say so and stop.

`DES` has no `Triage` and no cycle field that matters — `Backlog` is the live pool the designer draws from. There is no priority-encoded backlog order; read the real `priority` field (base § Priority).

### 2. Pick what to work on

The designer self-organizes — there's no PM hand-off here. Decision factors, in order:

- **Priority.** `Urgent` / `High` first; `Medium` is the bulk; `Low` only with slack.
- **Downstream impl pressure.** A Design Issue that `blocks` an Impl Issue the dev is about to pick up should jump the queue — its `Done` (set by the dev on `Dev approved`) is what releases the blocked Impl Issue (base § Blocking relations). If unsure which Impl Issue is queued next, check with the dev / PM in Slack rather than guessing.
- **Project arc.** Prefer advancing a Project already in flight over starting a new one's first macro.

If the user named a specific issue, skip the ranking and go straight to it. Otherwise state your pick + one-line rationale and let the user confirm or swap.

### 3. Self-assign + transition to `Todo`

This is the **one self-assign moment** in the designer's flow (self-assignment happens at `Backlog` → `Todo`). Pass `assignee: "me"` — the designer is the one running this skill. Don't hardcode a name.

Confirm in chat, then one write:

```
mcp__linear-server__save_issue
  id: "DES-N"
  assignee: "me"          # or the designer's resolved displayName
  state: "Todo"
```

`Todo` means committed-to-do, not yet started. The macro stays at `Todo` until the designer actually opens it.

### 4. Transition to `In Progress` on start

A separate moment — when the designer actually begins the macro. One write:

```
mcp__linear-server__save_issue
  id: "DES-N"
  state: "In Progress"
```

`In Progress` on `DES` means the designer is working on the macro (base § Statuses → `DES`). No label, no comment needed — the status change is the signal.

If the designer is claiming-and-starting in one sitting, steps 3 and 4 can be confirmed together (`Todo` then `In Progress`), but they're still two distinct states — don't skip `Todo`, the board reads it as "picked up but not opened yet".

### 5. Echo result in chat

- `DES-N` → new state.
- If the issue `blocks` an Impl Issue, note it: "this design blocks ENG-M; that Impl Issue stays blocked until this reaches `Done`".

## Notes

- **No cycles on `DES`.** Per ontology § 3 / § 4.2. There's no cycle to attach to and no PM-driven pull — the `Backlog` → `Todo` self-assign *is* the designer's commitment signal. Don't look for a `list_cycles` step here; that's the dev's `ENG` flow.
- **Self-assign is the only time the designer sets assignee.** Once assigned, the designer stays assignee through `Done` (Invariant 1, `events-tracker-base`). The dev's `In Dev Review` approval and the `Dev approved` label never change who's assigned.
- **Don't transition to `Done` from here.** `Done` on `DES` is the **dev's** action on `In Dev Review` → `Done` (sets `Dev approved`). The designer's forward path is `In Progress` → `In Dev Review` (see `submitting-design-for-dev-review.md`).
- **`Backlog` is fed by the PM.** If the Backlog is empty and there's design work the designer expects, that's a missing PM-created Design Issue — flag in Slack / to the PM rather than self-creating one.
