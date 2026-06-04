# Submitting a Design Issue for dev review

Recipe for the designer's hand-off: an own Design Issue in `EVTDES` `In Progress` is ready, transition it to `In Dev Review` and @-mention the responsible dev (the Project lead) so they sign off on implementability. Loaded on demand from `events-tracker-designer` → `SKILL.md` → § References pointer. Entry-point trigger: "отправь дизайн на ревью девам", "макет готов, на ревью", "submit design for dev review", "EVTDES-N ready for dev".

Out of scope: the dev's review decision (`In Dev Review` → `Done` / → `In Progress` — that's the dev's call, lives in the dev skill). Handling a rejection that comes back (see `handling-dev-rejection-of-design.md`). `EVTENG` review work (see `design-review-of-impl-issue.md`).

All steps run in chat. The transition + @-mention comment is one logical hand-off; confirm once before writing.

## Why `In Dev Review` exists

`EVTDES` runs a single review gate: the dev signs off that the macro is **implementable** before the designer's work is considered complete (base § Statuses → `EVTDES`). The dev — not the designer — flips the issue to `Done` and sets `Dev approved`. This step is the designer signaling "macro ready, your turn to check".

## Procedure

### 1. Confirm the issue is the designer's and is `In Progress`

```
mcp__linear-server__get_issue
  id: "EVTDES-N"
```

Check: `state` is `In Progress`, `assignee` is the designer. If it's still `Todo`, the designer hasn't formally started — that's fine, but the forward path is `Todo` → `In Progress` → `In Dev Review`; don't jump straight from `Todo`. If the assignee isn't the designer, stop and flag — this isn't the designer's issue to move (Invariant 1).

### 2. Resolve the responsible dev's `displayName`

The reviewer is the **Project lead** — the dev who owns the dependent Impl Issue (base § Map: Project lead = responsible dev). Resolve their Linear `displayName`:

- If the Design Issue is in a Project, read the Project's `lead`: `get_project name:"<Project name>"` (or follow `project` from `get_issue` in step 1).
- Otherwise resolve via `list_users name:"<dev name>"` (see `SKILL.md` → § Identity resolution).

Don't hardcode a dev name — the roster and Linear identities drift. If two devs and it's unclear which owns the impl, ask in one turn: "which dev reviews this — <X> or <Y>?".

### 3. Gather the design link + context for the comment

The macro lives in Figma, not in the tracker (brevity rule — base § Ontology → Brevity). The comment is where the designer points the dev at it:

- **Figma link** to the macro / frames under review.
- **One-line scope**: what flows / screens the macro covers, readable without opening Figma.
- **Open questions / risks** the dev should weigh on implementability, if any (e.g., "the live-update animation may be expensive — flag if it's a problem").

If the Figma link isn't in context, ask for it — don't transition without it; the dev can't review a macro they can't open.

### 4. Confirm, then transition + comment

Show the draft in chat: target state, the dev being @-mentioned, the comment body. Single confirmation, then:

```
# 1. Post the hand-off comment (@-mention the dev)
mcp__linear-server__save_comment
  issueId: "EVTDES-N"
  body: |
    @<dev displayName> macro ready for implementability review.

    Figma: <link>
    Scope: <one line — flows / screens covered>
    Notes: <open questions / risks, or "none">

# 2. Transition to In Dev Review
mcp__linear-server__save_issue
  id: "EVTDES-N"
  state: "In Dev Review"
```

`@-mention` syntax uses the dev's `displayName` (base § MCP-tool guide → `save_comment`). The comment body is **English** (Invariant 5) regardless of the chat language. The assignee stays the designer — no `assignee` field on this `save_issue` call (Invariant 1).

Comment-before-transition or transition-before-comment both work; keep them in the same confirmed batch so the dev's inbox notification (from the @-mention) lands with the status change.

### 5. Echo result in chat

- `EVTDES-N` → `In Dev Review`, dev @-mentioned.
- Note: the dev now owns the next move — approve (`→ Done`, sets `Dev approved`, releases any blocked Impl Issue) or reject (`→ In Progress` with a checklist, bounced back to the designer — handled in `handling-dev-rejection-of-design.md`).

## Notes

- **Designer never sets `Dev approved` or transitions to `Done`.** Both are the dev's actions on this issue (base § Labels → `EVTDES`; ontology § 4.2). The designer's job ends at `In Dev Review`; the next forward step is the dev's.
- **Assignee unchanged.** The dev is the reviewer, not the new owner — the @-mention + status change is the "you need to look" signal, no reassignment (Invariant 1, rationale in ontology § 4.3).
- **No labels on this transition.** `EVTDES` has only `Dev approved`, set by the dev later. The designer adds nothing.
- **If the dev approves**, the Design Issue reaches `Done` and — if it `blocks` an Impl Issue — releases that dependency (base § Blocking relations). The designer doesn't need to act further unless a new Design Issue is queued.
