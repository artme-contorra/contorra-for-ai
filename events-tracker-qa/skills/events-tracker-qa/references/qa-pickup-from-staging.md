# QA pickup — test an Impl Issue on staging

Recipe for QA's entry flow: an Impl Issue sits in `EVTENG` `QA` (the dev or designer @-mentioned QA after staging), read the issue + comments for context, then plan and run the test pass on the staging build. Loaded on demand from `events-tracker-qa` → `SKILL.md` → § References pointer. Entry-point trigger: "что мне на QA", "прогони EVTENG-N на стейджинге", "what's waiting for QA", "test this on staging".

Out of scope: the verdict itself — approve / reject / sub-issue bugs live in `qa-review-of-impl.md`. Issues in any status other than `QA` — QA only touches an Impl Issue on this review call (`SKILL.md` → § Role scope → What QA does NOT do). `EVTDES` Design Issues — not QA's board.

This step is all reads — no confirmation needed. The writes come at the verdict.

## Posture — why QA is the release gate

QA validates on staging **before** promotion to prod. There is no middle "ready to promote" status: `QA` → `Done` *is* the promotion, and `Done` means released (base § Statuses → `EVTENG`; ontology § 4.1). So QA's approve is the release gate — the sign-off that flips the build from staging to prod (delivery-flow § 2: "her sign-off is what flips a build from staging to prod ... without it, the build doesn't promote").

This is the transitional posture — delivery-flow § 7 step 0: QA is a **per-task phase**, sequential, running after design review on UI issues. In later steps it shifts to async sampling of staging batches, but that's substrate-gated and not where the team is today. Today: every `QA` issue is a task to test and gate, one at a time.

QA does not own the issue — the dev does, all the way to `Done` (Invariant 1). QA tests and decides; the assignee never changes.

## Procedure

### 1. Pull what's waiting for QA

```
mcp__linear-server__list_issues
  team: "Events Engineering"
  state: "QA"
  limit: 50
```

Show count + one line per entry: `EVTENG-N — title — assignee (dev) — Type label — UI?`. Or if the user named a specific `EVTENG-N`, `get_issue` it directly. If the queue is empty, say so and stop.

### 2. Read the issue for context

`get_issue EVTENG-N` and `list_comments issueId:"EVTENG-N"`. From these, pin down:

- **Staging link** — from the dev's (or designer's) hand-off @-mention comment. If absent, ask in a comment / Slack — **don't test blind.**
- **What changed** — the issue title + description (minimal for in-Project issues; context lives in the Project) and the hand-off comment. For an in-Project issue, the Project description carries the fuller "what / why".
- **Type label** — `bug` / `feature` / `chore` / `tech-debt` (base § Labels). A `bug` issue means "verify the fix + that it didn't regress nearby"; a `feature` means "exercise the new flow end-to-end".
- **`Design approved` state on UI issues** — see step 3.

### 3. Check the `Design approved` gate on UI issues

If the issue carries the `UI` label, it should already have the `Design approved` marker — the designer passes it on `Design Review` → `QA` before it reaches QA (ontology § 4.1; base Invariant 4). Confirm `Design approved` is present.

If a `UI` issue arrived in `QA` **without** `Design approved`, that's a flow violation — design review was skipped, not passed. Flag it to the dev / PM in a comment (@-mention the assignee) rather than silently testing — design fidelity is the designer's gate, not QA's, and a missing ack means the issue jumped a phase. Then hold the QA pass until it's resolved.

(A non-`UI` issue legitimately skips design review — the dev transitions `In Staging` → `QA` directly. No `Design approved` is expected there.)

### 4. Plan the test pass

From step 2, decide what to exercise on staging. Typical pass:

- **Happy path** — the primary flow the change enables / fixes, end to end on staging.
- **Edge / error cases** — empty states, invalid input, boundary conditions the change touches.
- **Regression around the change** — adjacent behaviour that could break (esp. on critical paths: auth, scoring, payments, registration, calendar).
- **Cross-surface** — if the change spans web behaviour the issue calls out, exercise each.
- **Acceptance against the ask** — does it actually do what the issue / Project description says it should.

State the plan in chat (one or two lines) so the user can add cases before you run.

### 5. Run on staging

Walk the plan on the staging build deliberately. Note each finding as you go — what, where, expected vs. actual, repro steps. These become the verdict input: clean → approve; deviations → either the rejection checklist or sub-issue bugs (see `qa-review-of-impl.md` for the bug-vs-checklist boundary).

### 6. Hand to the verdict

Summarise findings in chat — pass / fail per planned case — and move to `qa-review-of-impl.md` for the approve-or-reject decision and its writes.

## Notes

- **Don't test blind.** No staging link, no test. Ask the dev (the assignee) in a comment / Slack first.
- **QA is per-task today.** One issue, one full pass — not batch sampling (delivery-flow § 7 step 0). Don't approve a batch on a spot-check; that's the Step-1+ posture, not now.
- **Tester, not owner.** The assignee is the dev throughout. QA never self-assigns and never appears as assignee on the issue (Invariant 1).
- **Tracker language is English** (base Invariant 5) — any comment you post here (e.g., flagging a missing `Design approved`) is in English, even if the chat around it is Russian.
