# Handing off after staging

Recipe for the dev's hand-off: an own Impl Issue has reached `EVTENG` `In Staging` (merged + deployed to staging), route it to the right review phase — `Design Review` if it carries the `UI` label, else `QA` — with an @-mention to the reviewer. Loaded on demand from `events-tracker-dev` → `SKILL.md` → § References pointer. Entry-point trigger: "выложил на стейджинг", "отдай на ревью / QA", "hand off EVTENG-N", "it's on staging, who reviews".

Out of scope: getting to `In Staging` (you walk `In Progress` → `In Review` → `In Staging` manually until GitHub integration; the merge+deploy itself is off-tracker). The reviewers' decisions (designer / QA skills). Handling a bounce that comes back (see `responding-to-bounce.md`).

The comment + transition is one logical hand-off; confirm once before writing.

## Where you are

The work is merged and on staging — manually transitioned to `In Staging` for now (once the GitHub integration is on, the merge does this; base § Statuses → `EVTENG`). Before you hand off, you've already self-tested on staging as if no review phase existed (delivery-flow § 3) — the review phase validates, it doesn't first-detect. Now route it.

## Procedure

### 1. Read the issue: labels decide the route

```
mcp__linear-server__get_issue
  id: "EVTENG-N"
```

Confirm `state` is `In Staging` and `assignee` is you. Then inspect the labels — they decide where it goes:

- **`UI` present** → `Design Review` (the designer checks staging against the macro first; `QA` comes after).
- **No `UI`** → `QA` directly (no design surface to review).

The `UI` label is set by the PM at creation when the change touches the interface (base § Labels → `EVTENG`; Invariant 3). If you believe it's wrong — the change doesn't actually touch the UI, or it does and `UI` is missing — see § Disputing the `UI` label below before routing.

**Check for surviving marker labels (re-entry case).** If `Design approved` is already on the issue from a prior pass and the UI surface didn't change again on this pass, skip `Design Review` and go straight to `QA` even though `UI` is present — the design gate is already cleared and re-entry doesn't redo passed phases (base Invariant 4; ontology § 4.1). Same logic if `QA approved` survives and behaviour didn't change. State which phases you're skipping and why in the hand-off.

### 2a. `UI` present → `Design Review`

Resolve the designer's `displayName` (`list_users name:"<designer name>"`; don't hardcode). The staging link is mandatory — the designer can't compare against the macro without it. Confirm, then:

```
# 1. Hand-off comment, @-mention the designer, include the staging link
mcp__linear-server__save_comment
  issueId: "EVTENG-N"
  body: |
    @<designer displayName> on staging, ready for design review against the macro.

    Staging: <staging link>
    Notes: <anything to flag — known deviations, intentional choices, or "none">

# 2. Transition to Design Review
mcp__linear-server__save_issue
  id: "EVTENG-N"
  state: "Design Review"
```

Don't set any label here — `Design approved` is the designer's (base § Labels → `EVTENG`). Assignee stays you — no `assignee` field (Invariant 1).

### 2b. No `UI` → `QA`

Resolve QA's `displayName` (`list_users name:"<QA name>"`). Confirm, then:

```
# 1. Hand-off comment, @-mention QA, include the staging link
mcp__linear-server__save_comment
  issueId: "EVTENG-N"
  body: |
    @<QA displayName> on staging, ready for QA.

    Staging: <staging link>
    Flows to exercise: <the relevant user flows / what changed>

# 2. Transition to QA
mcp__linear-server__save_issue
  id: "EVTENG-N"
  state: "QA"
```

Don't set `QA approved` — that's QA's, set on `QA` → `Done` (base § Labels). Assignee stays you (Invariant 1).

### 3. Echo result in chat

- `EVTENG-N` → `Design Review` (designer @-mentioned) **or** → `QA` (QA @-mentioned), with the staging link included.
- Any phase you skipped via a surviving marker label, and why.
- Note the ball is now with the reviewer: approve (designer → `QA`; QA → `Done`) or bounce (`→ In Progress`, comes back to you — `responding-to-bounce.md`).

## Disputing the `UI` label

You can dispute the `UI` label on your own Impl Issue (base Invariant 3): comment your reasoning, @-mention the PM, then remove the label. `labels` on `save_issue` **replaces** the set, so pass the issue's existing labels minus `UI` (from step 1's `get_issue`) — don't wipe the `Type` label:

```
mcp__linear-server__save_issue
  id: "EVTENG-N"
  labels: ["<Type label>", "<other existing labels except UI>"]
```

If you remove `UI`, route to `QA`. Don't silently re-route around a `UI` you disagree with without removing the label — make the dispute visible (comment + @-mention PM) so the board stays truthful.

## Notes

- **You never transition to `Done` here.** Your hand-off ceiling is `Design Review` / `QA`. `Done` is QA's call after the `QA` phase (ontology § 4.1).
- **Staging link is non-negotiable on hand-off.** The whole point of `In Staging` is that the reviewer checks the deployed build — a hand-off without the link blocks them.
- **Marker labels survive bounces.** `Design approved` / `QA approved` from a prior pass mean the next `In Staging` skips that phase unless the relevant surface changed again (Invariant 4). That's the re-entry path from `responding-to-bounce.md`.
- **Never reassign.** Approve or bounce, the assignee stays you — you own it to `Done` (Invariant 1).
