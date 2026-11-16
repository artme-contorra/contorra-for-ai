# Picking up an Impl Issue

Recipe for the dev's entry flow: pull an Impl Issue assigned to you from `ENG` `Todo` in the active cycle into `In Progress`, after a short pre-flight. Loaded on demand from `events-tracker-dev` → `SKILL.md` → § References pointer. Entry-point trigger: "взять задачу в работу", "что мне взять", "start my next impl issue", "pull from Todo".

Out of scope: Triage sweep, cycle planning, pulling from `Backlog` into a cycle — those are the PM's (see `events-tracker-pm`). Design Issues you only *review* (see `dev-review-of-design.md`). Handing off after staging (see `handoff-after-staging.md`).

All steps run in chat. Transition to `In Progress` is one write; confirm once.

## Where work comes from

The PM plans the cycle: Impl Issues land in `Todo` attached to the active cycle, assigned to you. You don't self-pull from `Backlog` — that's the PM's cycle-planning move. Your entry point is `Todo` for the current cycle.

`In Progress` is yours to set on both systems. On 1.0 the later moves are manual too, as these plugins describe it. On 2.0 the moves after this one are Linear's GitHub integration's: a PR that names the issue moves it to `In Review`, the merge to `QA` (base § Statuses → ENG flow on 2.0).

## Procedure

### 1. Find what's assigned to you in the active cycle

First resolve the active cycle (it needs the team UUID, not the name — base § MCP-tool guide → `list_cycles` gotcha):

```
mcp__linear-server__list_cycles
  teamId: "23ad419d-cc6f-423d-a7f7-18cccb2833cc"   # ENG
  type: "current"
```

Then the `Todo` issues assigned to you in that cycle:

```
mcp__linear-server__list_issues
  team: "Events Engineering"
  state: "Todo"
  assignee: "me"
  cycle: "<current cycle number from above>"
  limit: 50
```

Show count + one-line per entry: `ENG-N — title — priority — project — blocks/blockedBy?`. If nothing is assigned to you in `Todo`, say so — don't reach into `Backlog` or another cycle; flag it to the PM instead.

### 2. Pick what to work on

If the user named a specific `ENG-N`, go straight to it. Otherwise rank by `priority` (`Urgent` / `High` first, `Medium` is the bulk, `Low` only with slack — base § Priority), then state your pick + one-line rationale and let the user confirm or swap.

### 3. Pre-flight

Before transitioning, `get_issue` the pick and confirm two things:

- **Blocking Design Issue is `Done`.** If the Impl Issue is `blockedBy` a Design Issue (`DES-N`), check that Design Issue's `state`. The `blocks` relation releases only when the design reaches `Done` (base § Blocking relations). If it's not `Done` yet, the macro isn't dev-approved — don't start impl against a moving target. Flag it to the designer / PM. (If you're the reviewer on that Design Issue and it's sitting in `In Dev Review`, do the review first — see `dev-review-of-design.md`.)
- **Project context understood.** In-Project Impl Issues carry minimal descriptions — the context lives in the Project description (base § Brevity rule). Read the Project (`get_project`) and the linked design doc / Figma macro so you're starting from the actual intent, not a one-line title. If there's no Project, the issue's own description should be self-contained.

You own this feature from the design phase to ship (delivery-flow § 3) — starting with the macro and context in hand is part of that ownership, not a formality.

### 4. Transition to `In Progress`

Confirm in chat, then one write. Assignee is already you — no `assignee` field needed.

```
mcp__linear-server__save_issue
  id: "ENG-N"
  state: "In Progress"
```

`In Progress` means branch / draft PR open, you're working (base § Statuses → `ENG`). No label or comment needed — the status change is the signal.

### 5. Echo result in chat

- `ENG-N` → `In Progress`.
- If a blocking Design Issue wasn't `Done`, report that you held off and why.

## Notes

- **You don't self-pull from `Backlog`.** Cycle planning (Backlog → Todo) is the PM's move (`events-tracker-pm` → `cycle-planning.md`). If `Todo` is empty and you have capacity, that's a planning gap — raise it, don't pull unplanned work into the cycle yourself.
- **Self-test posture starts now.** The PR self-test checklist and the design/QA phases downstream validate your work; they're not where bugs are first found (delivery-flow § 3). Build with that bar from `In Progress` onward.
- **What happens after `In Progress` depends on the system.** On 1.0 you walk the issue on by hand (`handoff-after-staging.md`). On 2.0 you don't touch its status again: the integration moves it, and the PR must name the `ENG-N` so it can.
