# Handling a dev rejection of a Design Issue

Recipe for the designer's bounce-back: an own Design Issue was sent to `In Dev Review`, the dev rejected it on implementability and transitioned it to `In Progress` with a checklist comment + @-mention. Parse the checklist, address the items, return to `In Dev Review`. Loaded on demand from `events-tracker-designer` → `SKILL.md` → § References pointer. Entry-point trigger: "дев вернул дизайн", "макет отклонили, поправить", "address the dev's design rejection", "EVTDES-N bounced back".

Out of scope: the original submission (see `submitting-design-for-dev-review.md`). The dev's review decision itself (dev skill). `EVTENG` review work (see `design-review-of-impl-issue.md`).

This mirrors the submission flow — same forward transition, after a fix pass. Confirm once before the re-submit batch.

## What happened

The dev reviewed the macro on `EVTDES` `In Dev Review`, judged it not yet implementable, posted a checklist of required changes, transitioned the issue back to `In Progress`, and @-mentioned the designer (base § Statuses → `EVTDES`; ontology § 4.2). The issue is the designer's own — assignee unchanged (Invariant 1) — so the designer just picks it back up and works the list.

## Procedure

### 1. Read the dev's checklist

```
mcp__linear-server__get_issue
  id: "EVTDES-N"

mcp__linear-server__list_comments
  issueId: "EVTDES-N"
```

Confirm: `state` is `In Progress`, `assignee` is the designer. Read the dev's most recent rejection comment — that's the checklist of what blocks implementability. Items are typically things like "this layout needs a fixed-width container the current grid can't express", "the animation timing isn't feasible with our component library", "missing the empty state for the list".

Summarize the checklist in chat so the user (designer) sees what's being addressed. If the checklist is ambiguous, ask the dev for clarification in a comment / Slack before reworking — don't guess at what "feasible" means.

### 2. Address the items

This is design work in Figma, off-tracker. The designer reworks the macro to resolve each checklist item. The issue stays `In Progress` throughout — no intermediate tracker state.

Track which items are resolved against the dev's checklist so the re-submission comment can speak to each one. If an item is **disputed** (the designer thinks the original was fine, or there's a better resolution than the dev proposed), that's a conversation in the comment thread / Slack — resolve it before re-submitting, not by silently ignoring the item.

### 3. Re-submit to `In Dev Review`

Once the checklist is addressed, return the issue to `In Dev Review` and @-mention the same dev with what changed. Resolve the dev's `displayName` (Project lead — from the Project `lead`, `knowledge.md` § Team, or `list_users`; don't hardcode). Confirm, then:

```
# 1. Re-submit comment, @-mention the dev, speak to the checklist
mcp__linear-server__save_comment
  issueId: "EVTDES-N"
  body: |
    @<dev displayName> addressed the review notes, ready for another look:

    - <item 1> — <how it was resolved>
    - <item 2> — <how it was resolved>
    - <item 3> — <resolved / discussed: outcome>

    Updated Figma: <link>

# 2. Transition back to In Dev Review
mcp__linear-server__save_issue
  id: "EVTDES-N"
  state: "In Dev Review"
```

Same shape as the original submission (`submitting-design-for-dev-review.md` step 4): comment-with-@-mention + transition, in one confirmed batch. Comment body is **English** (Invariant 5). Assignee stays the designer — no `assignee` field (Invariant 1). Include the updated Figma link so the dev re-reviews the current macro, not the stale one.

### 4. Echo result in chat

- `EVTDES-N` → `In Dev Review`, dev @-mentioned, checklist items addressed.
- Note the ball is back with the dev: approve (`→ Done`, sets `Dev approved`, releases any blocked Impl Issue) or another reject (back here again).

## Notes

- **The loop can repeat.** `In Dev Review` ⇄ `In Progress` cycles until the dev approves. Each bounce comes back here; each re-submit goes through step 3. No special handling on the Nth pass — same flow.
- **Designer never self-approves to `Done`.** No matter how many passes, the `In Dev Review` → `Done` transition (and `Dev approved`) is the dev's (ontology § 4.2). The designer's forward action is always `→ In Dev Review`.
- **Assignee unchanged across the whole loop.** The designer owns the Design Issue start to finish; the dev reviews via comment + @-mention + status change (Invariant 1).
- **Disputes are comments, not silent overrides.** If the designer disagrees with a checklist item, hash it out in the thread before re-submitting — re-submitting while ignoring a raised item just produces another bounce.
