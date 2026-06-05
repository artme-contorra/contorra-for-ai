# Dev review of a Design Issue

Recipe for the dev's review duty on `DES`: a Design Issue sits in `In Dev Review` (the designer @-mentioned the dev after the macro was ready), review the macro for **implementability**, then approve to `Done` or reject to `In Progress`. Loaded on demand from `events-tracker-dev` → `SKILL.md` → § References pointer. Entry-point trigger: "проверь макет на реализуемость", "дизайнер скинул на ревью", "review the design for implementability", "DES-N in dev review".

Out of scope: own Impl Issues in `ENG` (see the other refs). The designer's authoring and rework — those are the designer's flow. Moving a Design Issue in any status other than `In Dev Review` — the dev only touches it on this review call (`SKILL.md` → § Role scope → What the dev does NOT do).

This is the dev's only write path on `DES`, and it's the **one place the dev transitions anything to `Done`**. The `In Dev Review` → `Done` transition is per-call confirmation (`SKILL.md` → § Confirmation rules).

## Why the dev is here

`DES` runs a single review gate: the dev signs off that the macro is **implementable** before the designer's work is considered complete (base § Statuses → `DES`; ontology § 4.2). The designer transitioned the issue to `In Dev Review` and @-mentioned the dev. The dev — not the designer — flips it to `Done` and sets `Dev approved`. The designer owns the Design Issue all the way to `Done` (Invariant 1); the dev reviews via comment + @-mention + the status decision, and never reassigns.

## Procedure

### 1. Read the macro + the designer's hand-off

```
mcp__linear-server__get_issue
  id: "DES-N"

mcp__linear-server__list_comments
  issueId: "DES-N"
```

Confirm `state` is `In Dev Review` and `assignee` is the designer (stays the designer throughout — Invariant 1). Read the designer's submission comment for the Figma link, the scope, and any flagged risks. The macro lives in Figma, not the tracker (base § Brevity rule) — open it; if the link is missing, ask the designer in a comment / Slack rather than reviewing blind.

### 2. Review for implementability

You're judging whether the macro can be built with the team's stack and components — not whether it's pretty (that's the designer's domain). Typical checklist:

- **Component feasibility** — the layout / controls map to the existing component library, or the new component is scoped and reasonable.
- **Layout / responsive** — the grid and breakpoints are expressible; no fixed-width assumptions the responsive system can't honor.
- **States covered** — empty / loading / error / disabled states the impl will need are specified, not left implicit.
- **Interaction / animation** — transitions and timing are feasible (e.g., the live-update animation isn't prohibitively expensive).
- **Data / API reality** — the macro doesn't assume data or latency the backend can't deliver.
- **Edge content** — long strings, missing avatars, large lists, i18n where relevant.

Note each gap as you go — these become the rejection checklist.

### 3a. Approve → `Done` (sets `Dev approved`, releases the block)

If the macro is implementable (gaps negligible or already discussed), approve. This is per-call confirmation. Note the consequence before confirming: **transitioning the Design Issue to `Done` releases its `blocks` relation, unblocking the dependent Impl Issue** (base § Blocking relations) — say so in chat so the user knows the downstream Impl Issue just opened up.

`labels` on `save_issue` **replaces** the set — but `DES` has only the `Dev approved` label, so unless the designer added others, `labels: ["Dev approved"]` is the full set. Confirm, then:

```
# 1. Approve comment, @-mention the designer
mcp__linear-server__save_comment
  issueId: "DES-N"
  body: |
    @<designer displayName> implementability review passed — macro is buildable as specified. Marking Dev approved.

# 2. Set Dev approved label + transition to Done
mcp__linear-server__save_issue
  id: "DES-N"
  labels: ["Dev approved"]
  state: "Done"
```

`Dev approved` is Title case, set by the dev (base § Labels → `DES`). Assignee stays the designer — no `assignee` field (Invariant 1). Resolve the designer's `displayName` via `list_users name:"<designer name>"`; don't hardcode.

### 3b. Reject → `In Progress`

If the macro isn't yet implementable, reject with a concrete checklist of what blocks it and bounce back to the designer.

```
# 1. Checklist comment, @-mention the designer (the assignee)
mcp__linear-server__save_comment
  issueId: "DES-N"
  body: |
    @<designer displayName> implementability review — changes needed before this is buildable:

    - [ ] <blocker 1 — what, where, why it's not feasible / what's missing>
    - [ ] <blocker 2>
    - [ ] <blocker 3>

# 2. Transition back to In Progress
mcp__linear-server__save_issue
  id: "DES-N"
  state: "In Progress"
```

Do **not** set `Dev approved` on a reject. Don't transition to `Done`. Don't reassign — the designer owns it (Invariant 1). The designer addresses the checklist and re-submits to `In Dev Review` (their bounce-handling flow), and you review again. Make checklist items concrete and feasibility-grounded — "this animation timing isn't feasible with our component library", not "feels off".

### 4. Echo result in chat

- `DES-N` → `Done` (approved, `Dev approved` set; the dependent Impl Issue is now unblocked — name it if you know it) **or** → `In Progress` (rejected, checklist posted, designer @-mentioned).

## Notes

- **`Done` here is the dev's one `Done` transition.** Approving a design is the only place the dev moves anything to `Done`. The dev never transitions an own *Impl* Issue to `Done` — that's QA's call after the QA phase (ontology § 4.1, § 4.2).
- **`Dev approved` is the dev's label; `Design approved` / `QA approved` are not.** `Dev approved` lives on `DES`; the `ENG` markers belong to the designer and QA (base § Labels). Don't confuse them.
- **The review can loop.** `In Dev Review` ⇄ `In Progress` cycles until the macro is implementable. Each pass comes back here; same flow, no special handling on the Nth pass.
- **Never reassign.** Approve or reject, the assignee stays the designer (Invariant 1). The @-mention + status change is the signal.
- **Comment in Figma during design, too.** The iterative dev↔design loop is expected (delivery-flow § 3) — catching feasibility gaps early in Figma beats catching them on the formal `In Dev Review` pass.
