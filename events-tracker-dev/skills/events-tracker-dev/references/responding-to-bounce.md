# Responding to a review bounce

Recipe for the dev's bounce-back: an own Impl Issue was sent back to `EVTENG` `In Progress` from `Design Review` (designer) or `QA` (QA reviewer) with a checklist comment + @-mention. Parse the checklist, fix it, walk the issue forward again, and hand off — skipping any phase whose marker label already survives. Loaded on demand from `events-tracker-dev` → `SKILL.md` → § References pointer. Entry-point trigger: "вернули задачу, поправить", "QA / дизайн отбили", "address the review bounce", "EVTENG-N bounced back".

Out of scope: the reviewers' decisions themselves (designer / QA skills). The first-time hand-off (see `handoff-after-staging.md`). Picking up fresh work (see `picking-up-impl-issue.md`).

## What happened

A reviewer judged the staging build short of the bar, posted a checklist of what to fix, transitioned the issue back to `In Progress`, and @-mentioned you. The issue is yours — assignee unchanged (Invariant 1) — so you pick it back up and work the list. The reviewer's surviving marker label (`Design approved` / `QA approved`) records which gates are already cleared (base Invariant 4).

## Procedure

### 1. Read the bounce + the surviving markers

```
mcp__linear-server__get_issue
  id: "EVTENG-N"

mcp__linear-server__list_comments
  issueId: "EVTENG-N"
```

Confirm `state` is `In Progress`, `assignee` is you. Read the reviewer's most recent comment — that's the checklist. Note which **marker labels** are present:

- `Design approved` present → the design gate is already cleared. This bounce came from `QA` (a Design-Review bounce would not carry `Design approved` — the designer only sets it on approve). On the next `In Staging` you skip `Design Review` and go straight to `QA` **unless your fix changes the UI surface again** (base Invariant 4; ontology § 4.1).
- `QA approved` present → behaviour already passed QA on a prior pass; skip `QA` on re-entry unless behaviour changed again. (Rare — usually means a Design-Review bounce after QA already passed.)
- Neither → this is a Design-Review bounce on a `UI` issue with no prior approvals; walk the full path.

Summarize the checklist + which phases are pre-cleared in chat so the user sees the plan. If a checklist item is ambiguous, ask the reviewer in a comment / Slack before reworking — don't guess. If you **disagree** with an item, raise it in the thread (it's a conversation), don't silently ignore it — ignoring a raised item just earns another bounce.

### 2. Check for sub-issue bugs blocking the parent

A review bounce may have produced **sub-issue bugs** on your Impl Issue rather than (or alongside) checklist lines — a gap big enough to be its own tracked item (base § Ontology → Sub-issue, use 1). Those sub-issues `block` the parent, and the parent can't move forward until they reach `Done`:

```
mcp__linear-server__list_issues
  team: "Events Engineering"
  parentId: "EVTENG-N"
```

Resolve each sub-issue (the same fix work) and walk it to `Done` as part of this bounce — the parent stays held until they close.

### 3. Fix it

Off-tracker dev work: address each checklist item and any sub-issue bug, then push the fix (branch / PR — your team's PR conventions, with the self-test checklist filled). The issue stays `In Progress` throughout — no intermediate tracker state. Self-test the fix on staging as if the review phase didn't exist (delivery-flow § 3); the re-review validates, it shouldn't be where the same class of bug is caught twice.

Track which items are resolved so the re-hand-off comment can speak to each one.

### 4. Walk forward and hand off, skipping cleared phases

Once merged + deployed to staging, transition back to `In Staging` (manual until GitHub integration; base § Statuses → `EVTENG`):

```
mcp__linear-server__save_issue
  id: "EVTENG-N"
  state: "In Staging"
```

Then hand off per `handoff-after-staging.md`, but **route based on the surviving markers, not just the `UI` label** (base Invariant 4):

- **Design-Review bounce, no surviving markers** (`UI` issue, design gate not yet cleared) → back to `Design Review`, @-mention the designer with the staging link. They re-check the surface they bounced.
- **QA bounce with `Design approved` surviving**, and your fix didn't touch the UI again → skip `Design Review`, go straight to `QA`, @-mention QA. Note in the comment that `Design approved` survives and the surface didn't change.
- **QA bounce where your fix *did* re-touch the UI surface** → re-route through `Design Review` first even though `Design approved` survives — a changed surface needs a fresh design look. State why.

The hand-off comment must speak to the checklist — what each item's fix was — and carry the staging link. Single confirmed batch (comment + transition), same shape as `handoff-after-staging.md`. Assignee stays you; you set no marker labels (those are the reviewers').

### 5. Echo result in chat

- `EVTENG-N` → `In Staging` → `Design Review` / `QA` (reviewer @-mentioned), with which phases were skipped and why.
- Any sub-issue bugs resolved to `Done`.
- Note the ball is back with the reviewer: approve forward (designer → `QA`; QA → `Done`) or bounce again (back here).

## Notes

- **Marker labels are the memory of cleared gates.** `Design approved` / `QA approved` survive bounces by design (Invariant 4) — they're what lets re-entry skip a passed phase. Don't strip them, and don't re-run a gate they've already cleared unless the relevant surface / behaviour actually changed again.
- **The loop can repeat.** A re-bounce comes right back here; same flow, no special handling on the Nth pass.
- **You never set a marker or hit `Done`.** Your forward action is always *into* a review state (`Design Review` / `QA`); the reviewer sets their label and makes the approve / `Done` call (ontology § 4.1).
- **A re-touched UI surface re-opens design review.** Surviving `Design approved` only buys a skip when the surface didn't change again — if your QA fix moved the UI, give the designer a fresh look (Invariant 4).
- **Never reassign.** You own the Impl Issue through every bounce to `Done` (Invariant 1).
