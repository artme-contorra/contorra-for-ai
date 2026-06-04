# QA verdict on an Impl Issue

Recipe for QA's verdict: after testing a `QA` issue on staging (`qa-pickup-from-staging.md`), approve to `Done` or reject to `In Progress`. Loaded on demand from `events-tracker-qa` → `SKILL.md` → § References pointer. Entry-point trigger: "проверил, всё ок, закрывай EVTENG-N", "approve and release", "qa reject EVTENG-N", "верни на доработку".

Out of scope: the test pass itself (see `qa-pickup-from-staging.md`). Issues in any status other than `QA` — QA only touches an Impl Issue on this call (`SKILL.md` → § Role scope → What QA does NOT do). `EVTDES` and the dev's / designer's own approvals.

This is QA's only write path on the board. Confirm once for a reject batch. **Confirm the `Done` transition (approve) per call, every time** — it releases to prod.

## Why this is the release gate

QA's approve sets `QA approved` and transitions `QA` → `Done`. `Done` means **released to prod** — there is no intermediate "ready to promote" status (base § Statuses → `EVTENG`; ontology § 4.1). The promotion itself happens outside the tracker (deploy / flip from staging); the `Done` transition records it. So the approve is a real release decision, not a stamp (delivery-flow § 2) — which is why it carries per-call confirmation.

QA does not own the issue. The assignee is the dev, all the way to `Done` (Invariant 1). Approve or reject, the assignee never changes.

## 1a. Approve → `Done`

If staging holds up across the planned pass (`qa-pickup-from-staging.md` step 4), approve. This sets the `QA approved` marker and releases to prod.

The dev to thank / notify is the **assignee** — already on the issue from the pickup's `get_issue` (no lookup needed). Confirm the release explicitly (this is the `Done` call), then:

```
# 1. Approve comment
mcp__linear-server__save_comment
  issueId: "EVTENG-N"
  body: |
    QA passed on staging — <one line: flows exercised, all green>. Approving for release.

# 2. Set QA approved label + transition to Done
mcp__linear-server__save_issue
  id: "EVTENG-N"
  labels: ["<existing labels>", "QA approved"]
  state: "Done"
```

`labels` on `save_issue` **replaces** the set — pass the issue's existing labels (the `Type` label, `UI`, `Design approved`, etc., from the pickup's `get_issue`) **plus** `QA approved`, or you'll wipe them (base § MCP-tool guide → `save_issue` — update). `QA approved` is Title case, set by QA (base § Labels → `EVTENG`); it survives later bounces (Invariant 4 — if the issue ever reopens, the next pass skips QA unless behaviour changed; QA doesn't re-clear it). The assignee stays the dev — no `assignee` field (Invariant 1).

## 1b. Reject → `In Progress`

If staging fails the pass, reject with a concrete checklist and bounce back to the dev.

```
# 1. Checklist comment, @-mention the dev (the assignee)
mcp__linear-server__save_comment
  issueId: "EVTENG-N"
  body: |
    @<dev displayName> QA — issues found on staging, needs another pass:

    - [ ] <finding 1 — what, where, expected vs. actual, repro>
    - [ ] <finding 2>
    - [ ] <finding 3>

    Staging: <link tested>

# 2. Transition back to In Progress
mcp__linear-server__save_issue
  id: "EVTENG-N"
  state: "In Progress"
```

The `<dev displayName>` is the assignee — @-mention by `displayName` (resolve via `list_users name:"<dev name>"` only if it isn't already to hand). Do **not** set `QA approved` on a reject. Don't reassign — the dev owns it (Invariant 1). The dev addresses the checklist and walks the issue forward again; `Design approved` (if present) survives, so a non-UI fix can come back straight to `QA` (Invariant 4).

## 2. Out-of-scope findings → new Triage issue; sub-issue only in extreme cases

**In-scope findings belong in the rejection checklist (1b)** — that's the default. Two non-default cases:

**Out-of-scope finding** (broken behaviour noticed while testing that isn't part of this issue's scope): create a **new standalone `EVTENG` issue** — no `parentId`, doesn't block the parent, doesn't hold the verdict.

```
mcp__linear-server__save_issue
  team: "Events Engineering"
  title: "<the finding — standalone-readable>"
  description: "<repro / expected vs. actual / staging reference>"
  labels: ["bug"]
```

A QA-created `EVTENG` issue lands in `Triage` (base Invariant 2 — only the PM skips it); the PM routes it at sweep. `description` is **required** and **English** (Invariant 5). An out-of-scope finding alone is not a reason to reject — if the issue's own scope passes, approve and file the finding separately.

**Extreme case — in-scope gap that must be its own tracked item** while still blocking the parent: same call but with `parentId: "EVTENG-N"` (base § Ontology → Sub-issue, use a). The parent stays in `QA` (blocked) until the sub-issue reaches `Done` — don't approve a parent with open QA sub-issues. @-mention the dev (assignee) and PM with the new `EVTENG-N` so the block is visible before the sweep. This is rare — reach for the checklist first.

## 3. Echo result in chat

- `EVTENG-N` → `Done` (approved, `QA approved` set, **released to prod**) **or** → `In Progress` (rejected, checklist posted, dev @-mentioned).
- Any out-of-scope findings filed to `Triage` (their `EVTENG-N`); in the rare sub-issue case, a note that the parent stays in `QA` until it closes.

## Notes

- **Approve = released to prod.** The `Done` transition is the release gate, not a workflow nicety. Per-call confirmation, always (`SKILL.md` → § Confirmation rules). Never batch it.
- **`QA approved` is QA's label; `Design approved` and `Dev approved` are not.** `Design approved` is the designer's (on `Design Review`); `Dev approved` is the dev's (on `EVTDES` `In Dev Review`). On `EVTENG`, QA sets only `QA approved` (base § Labels).
- **Marker labels survive bounces.** `QA approved` (and any `Design approved` already present) persist across a reopen; the next pass skips the cleared phase unless the relevant surface actually changed (Invariant 4). QA doesn't re-clear them.
- **Never reassign on either branch.** Approve or reject, the assignee stays the dev (Invariant 1). The @-mention + status change is the signal.
- **English only** (Invariant 5) — comments and any Triage-arriving issue content, regardless of chat language.
