# Design review of an Impl Issue on staging

Recipe for the designer's review duty on `EVTENG`: an Impl Issue sits in `Design Review` (the dev @-mentioned the designer after staging), compare the staging build against the Figma macro, then approve to `QA` or reject to `In Progress`. Loaded on demand from `events-tracker-designer` → `SKILL.md` → § References pointer. Entry-point trigger: "проверь impl на стейджинге", "design review для EVTENG-N", "review the implementation against the macro", "что мне на дизайн-ревью".

Out of scope: own Design Issues in `EVTDES` (see the other refs). The dev's bounce-handling and QA's own review — those are other roles' flows. Moving an Impl Issue in any status other than `Design Review` — the designer only touches it on this review call (`SKILL.md` → § Role scope → What the designer does NOT do).

This is the designer's only write path on `EVTENG`. Confirm once for the approve/reject batch.

## Why the designer is here

When an Impl Issue carries the `UI` label, the dev transitions it `In Staging` → `Design Review` and @-mentions the designer (base § Statuses → `EVTENG`; ontology § 4.1). The designer checks that what shipped to staging **matches the macro** before it goes to QA. The designer does not own the Impl Issue — the dev does, all the way to `Done` (Invariant 1).

## Procedure

### 1. Pull what's waiting for design review

```
mcp__linear-server__list_issues
  team: "Events Engineering"
  state: "Design Review"
  limit: 50
```

Show count + one-line per entry: `EVTENG-N — title — assignee (dev) — project`. Or if the user named a specific `EVTENG-N`, `get_issue` it directly. Read the dev's hand-off comment for the staging link and any notes: `list_comments issueId:"EVTENG-N"`.

### 2. Gather staging build + macro

- **Staging link** — from the dev's @-mention comment. If absent, ask the dev in a comment / Slack; don't review blind.
- **Figma macro** — the reference. The source of truth is the macro that was `Dev approved` on the corresponding Design Issue (follow the Project / `blockedBy` link to find it).

### 3. Compare staging against the macro

Walk the implemented UI against the macro deliberately. Typical checklist:

- **Layout / spacing** — alignment, padding, sizing match the macro.
- **Typography** — sizes, weights, line-heights.
- **Color / tokens** — correct palette, states (hover / active / disabled).
- **Copy** — text matches; no placeholder strings.
- **States** — empty / loading / error states present where the macro specifies them.
- **Responsive** — behaves at the breakpoints the macro covers.
- **Interaction** — transitions / animations as designed.

Note each deviation as you go — in-scope ones become the rejection checklist; out-of-scope findings become standalone `Triage` issues (step 5).

### 4a. Approve → `QA`

If the impl matches the macro (or deviations are negligible and intentional), approve. This sets the `Design approved` marker and hands off to QA — it does **not** go to `Done` (that's QA's call after the `QA` phase).

Resolve QA's `displayName` via `list_users name:"<QA name>"` (see `SKILL.md` → § Identity resolution). Confirm, then:

```
# 1. Approve comment, @-mention QA
mcp__linear-server__save_comment
  issueId: "EVTENG-N"
  body: |
    Design review passed — staging matches the macro. @<QA displayName> over to you for QA.

# 2. Set Design approved label + transition to QA
mcp__linear-server__save_issue
  id: "EVTENG-N"
  labels: ["<existing labels>", "Design approved"]
  state: "QA"
```

`labels` on `save_issue` **replaces** the set — pass the issue's existing labels (the `Type` label, `UI`, etc., from step 1's `get_issue`) plus `Design approved`, or you'll wipe them (base § MCP-tool guide → `save_issue` — update). `Design approved` is Title case, set by the designer (base § Labels → `EVTENG`); it survives later bounces. The assignee stays the dev — no `assignee` field (Invariant 1).

### 4b. Reject → `In Progress`

If the impl diverges from the macro, reject with a concrete checklist and bounce back to the dev.

```
# 1. Checklist comment, @-mention the dev (the assignee)
mcp__linear-server__save_comment
  issueId: "EVTENG-N"
  body: |
    @<dev displayName> design review — changes needed before QA:

    - [ ] <deviation 1 — what, where, expected per macro>
    - [ ] <deviation 2>
    - [ ] <deviation 3>

    Macro: <Figma link>

# 2. Transition back to In Progress
mcp__linear-server__save_issue
  id: "EVTENG-N"
  state: "In Progress"
```

The dev (assignee) is @-mentioned. Do **not** set `Design approved` on a reject. Don't reassign — the dev owns it. The dev addresses the checklist and re-enters the flow (their `responding-to-bounce` flow); `Design approved` only gets set when the impl actually passes.

### 5. Out-of-scope findings → new Triage issue; sub-issue only in extreme cases

**In-scope deviations belong in the rejection checklist (4b)** — that's the default. Two non-default cases:

**Out-of-scope finding** (a bug or gap noticed during review that isn't part of this issue's scope): create a **new standalone `EVTENG` issue** — no `parentId`, doesn't block the parent, doesn't hold the review.

```
mcp__linear-server__save_issue
  team: "Events Engineering"
  title: "<the finding — standalone-readable>"
  description: "<repro / expected vs actual / where it was noticed>"
  labels: ["bug"]
```

A designer-created `EVTENG` issue lands in `Triage` (base Invariant 2 — only the PM skips it); the PM routes it at sweep. Description is **required** and **English** (Invariant 5).

**Extreme case — in-scope gap that must be its own tracked item** while still blocking the parent: same call but with `parentId: "EVTENG-N"` (base § Ontology → Sub-issue, use a). The parent stays in its review phase until the sub-issue reaches `Done`; @-mention the dev and PM in the review comment with the new `EVTENG-N` so the block is visible before the sweep. This is rare — reach for the checklist first.

### 6. Echo result in chat

- `EVTENG-N` → `QA` (approved, `Design approved` set, QA @-mentioned) **or** → `In Progress` (rejected, checklist posted, dev @-mentioned).
- Any out-of-scope findings filed to `Triage` (their `EVTENG-N`); in the rare sub-issue case, a note that the parent stays blocked until it closes.

## Notes

- **Approve goes to `QA`, never `Done`.** The designer's gate is design-fidelity; QA's gate (and the `Done` transition) comes after (ontology § 4.1). The designer never transitions an Impl Issue to `Done`.
- **`Design approved` is the designer's label; `Dev approved` is not.** Don't confuse the two — `Dev approved` lives on `EVTDES` and is the dev's (base § Labels). On `EVTENG` the designer sets `Design approved`, QA sets `QA approved`.
- **Marker labels survive bounces.** If `Design approved` is already on the issue from a prior pass and only non-UI behaviour changed, the dev may skip `Design Review` on the next `In Staging` (Invariant 4). The designer doesn't re-clear it.
- **Never reassign on either branch.** Approve or reject, the assignee stays the dev (Invariant 1). The @-mention + status change is the signal.
