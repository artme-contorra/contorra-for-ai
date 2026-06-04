# Cycle planning on `EVTENG`

Recipe for the weekly PM flow: pull Impl Issues from `EVTENG` `Backlog` to `Todo` for the active cycle, attach them to the cycle, sanity-check priorities and assignees. Loaded on demand from `events-tracker-pm` → `SKILL.md` → § References pointer. Entry-point trigger: "запланируй цикл events", "events cycle planning", "что пулим в текущий цикл".

Out of scope: `EVTDES` (no cycles — designer self-organizes from `Backlog`). Mid-cycle re-planning (use Triage sweep + per-entry edit instead). Cycle creation / closing (Linear handles automatically; manual rewrites are a destructive op — see § Notes).

All steps run in chat. Single confirmation in step 5 covers the batch.

## Procedure

### 1. Identify the active cycle

`list_cycles` requires `teamId` (UUID), **not** team key or name. Use the `EVTENG` UUID from `SKILL.md` → § Map → Linear teams:

```
mcp__linear-server__list_cycles
  teamId: "23ad419d-cc6f-423d-a7f7-18cccb2833cc"
  type: "current"
```

Expected return: one cycle (or `[]` if cycles aren't running yet — flag in chat and stop). Capture the cycle's number (`"<n>"`) — that's what `save_issue` `cycle` takes.

If the user asks about a future cycle, pass `type: "next"`. For the historical view, `type: "previous"`.

### 2. Pull the `EVTENG` Backlog

```
mcp__linear-server__list_issues
  team: "Events Engineering"
  state: "Backlog"
  orderBy: "updatedAt"
  limit: 100
```

Show count and a one-line summary per entry: `EVTENG-N — title — priority — assignee — project`. Sort the displayed list by priority (Urgent first, then High, then Medium, then Low, then None), then by Project (group neighbors so the user sees cross-Project balance).

### 3. Propose the cycle slate

Walk the Backlog and propose a slate. Decision factors, in order:

- **Priority.** Urgent + High enter the slate by default unless capacity argues against. Medium fills remaining capacity. Low only if there's slack.
- **Dev balance.** Aim for a roughly even split across `EVTENG` devs based on capacity. If one dev is on vacation or has a heavy carry-over from last cycle, shift the balance.
- **Project completion arc.** If a Project has Impl Issues across Backlog and the previous cycle is partway done, prefer to advance it over starting a new Project's first Issue. Don't fan out across too many Projects in one cycle.
- **`blockedBy` state.** Skip Impl Issues whose `blockedBy` Design Issue isn't `Done` yet — they'll be released by the Design Issue closing, not by being pulled into the cycle. Surface as "blocked, leaving in Backlog: EVTENG-N (waiting on EVTDES-M)".

State the proposed slate in chat with a one-line rationale per entry. Ask the user to confirm, swap, or drop entries.

### 4. Sanity-check per entry

For each entry in the proposed slate, confirm:

- **Priority still right?** If something has aged in Backlog at the wrong priority, this is the moment to fix it.
- **Assignee still right?** If a Project changed lead or a dev capacity changed, reassign here (changing the actual owner is a legitimate use of reassignment — distinct from Invariant 1 which is about review-handoff reassignment).
- **`UI` label still right?** If the work has crystallized since creation and the UI scope changed, adjust.
- **Cycle number correct?** Usually the active cycle, but for high-priority items the user may pre-commit to `next` instead.

If anything needs adjustment, capture it; the batch in step 5 covers per-entry edits.

### 5. Execute the batch

For each entry in the confirmed slate:

```
mcp__linear-server__save_issue
  id: "EVTENG-N"
  state: "Todo"
  cycle: "<cycle-number>"
  priority: <adjusted if needed>
  assignee: "<adjusted if needed>"
  labels: ["<existing labels, adjusted if needed>"]
```

Pass `cycle` by number (e.g., `"7"`) — or by ID if the team has cycle names. Use the value captured in step 1.

`labels` on `save_issue` replaces the set. Pass the existing label set to keep them; only modify what changed. To remove an issue from a cycle, pass `cycle: null`.

If a sub-issue blocker surfaced during the sanity-check (e.g., "needs EVTENG-X to land first"), set `blockedBy` on the entry — append-only via `save_issue`, see `SKILL.md` → § MCP-tool guide → `save_issue` — update. The dependent issue stays in `Backlog` until the blocker closes.

### 6. End-of-planning summary

Print the final slate in chat: count pulled, count by dev, count by Project, count by priority. Surface the count of Backlog entries that stayed (deferred to next cycle or blocked).

If the team will discuss the slate at the next planning meeting, suggest copying the summary into the meeting transcript or `#team-events-ru` post-discussion.

## Notes

- **`EVTENG` runs weekly cycles, `EVTDES` runs none.** Per `2026-06-04_events-tracker-ontology-design.md` § 3. Don't try to pull Design Issues into a cycle — they have no cycle field that matters and the designer self-organizes from `Backlog` via the `Backlog` → `Todo` self-assign flow.
- **GitHub integration is deferred.** Status transitions on the Impl Issue (`Todo` → `In Progress` on branch open, etc.) are currently manual on the dev side. The cycle planning here only sets `Todo`; the dev moves to `In Progress` when they start.
- **Cycle rewrites are destructive.** Re-attaching multiple in-progress issues across cycles (e.g., "move everything not done from cycle 5 to cycle 6") is a bulk destructive op — per-call confirmation per `SKILL.md` → § Role scope and guard-rails → Confirmation rules. Prefer letting Linear auto-roll-over open issues at cycle boundary; only manually rewrite when there's a real planning reason.
- **`list_cycles` gotcha — UUID, not team key.** Documented in `SKILL.md` → § MCP-tool guide → `list_cycles` gotcha; restated here because step 1 is where you'll hit it. Passing `"EVTENG"` or `"Events Engineering"` to `teamId` returns an error.
- **Empty cycle list.** If `list_cycles` returns `[]`, cycles aren't enabled or there's no active cycle window. Don't fake one — surface in chat. The user may need to enable cycles in Linear's team settings (UI-only) before planning works.
