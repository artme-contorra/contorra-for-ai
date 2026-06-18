# events-tracker-public — design

**Date:** 2026-06-18
**Status:** approved (brainstorming) — pending implementation plan

## Purpose

A new, self-contained Claude Code plugin for people **outside** the Events dev
team — leadership and stakeholders (CEO, PM director, product owners) who are not
part of day-to-day development. It gives them two capabilities over the Events
tracker (Linear) through Claude, **without ever touching or learning the team's
internal workflow**:

1. **FILE a request into ENG Triage** (the *primary, most frequent* use case) —
   turn a plain-language description into a well-formed issue (clear English
   title, actionable description) landed in the `Triage` status of team `ENG`
   (Events Engineering), where the team picks it up.
2. **READ the tracker** (read-only, across the whole Events product — `ENG` +
   `DES`) — status of an issue, what's happening now, find an issue by topic,
   summarize a project, exec-level overview of initiatives and projects.

## Why a separate plugin (not a shared-base split)

The existing suite (`events-tracker-shared` + role plugins PM/designer/dev/QA)
puts catalog + ontology + **internal process** (invariants, label/blocking
mechanics, review choreography) into one shared base. An outside consumer needs
**none** of that process — and ideally sees as little of it as possible.

Decision: **standalone plugin, no dependency on `events-tracker-shared`**, with
its own bundled Linear MCP and a deliberately **thin** ontology written at
exec/contributor altitude. Rejected alternatives:

- **Split `shared` into `ontology-core` + `process-layer`:** the base has no
  clean ontology/process seam at the line we need — statuses, labels, blocking,
  sub-issues, title/brevity rules are all process-flavored. Even a clean
  `ontology-core` would over-share to the outsider. Plus it touches `shared` +
  all 4 role plugins (version bumps, breakage risk) for one new, read-mostly
  consumer. The duplication it removes is small and stable.
- **New public base + role on top:** premature (YAGNI) — one outside consumer
  today. Revisit only if a second external role appears.

The outsider's knowledge is a **curation, not a subset** of the base — a
genuinely smaller, differently-framed artifact. Duplication of the stable
entity/status vocabulary is cheap and guarded by a one-line drift note (below).

## Plugin

- **Name:** `events-tracker-public` (skill also `events-tracker-public`).
- **Audience:** leadership / stakeholders outside development.
- **Dependencies:** none. Installed directly and alone (no role plugin
  alongside), so its bundled MCP does not double-register.

```
events-tracker-public/
  .claude-plugin/plugin.json        # v0.1.0, no dependencies
  .mcp.json                         # copy of shared: mcp-remote -> https://mcp.linear.app/mcp
  skills/events-tracker-public/
    SKILL.md                        # leads with filing (inline) + thin ontology + pointer to reading
    references/
      reading-the-tracker.md        # read recipes, loaded on demand
```

`.mcp.json` is a verbatim copy of `events-tracker-shared/.mcp.json` (stdio
transport via `mcp-remote`, per the repo's MCP rule).

## Thin read layer (ontology) — in SKILL.md

Deliberately minimal — exec altitude, not the full 11+7 status tables:

- **Teams:** `ENG` (Events Engineering) and `DES` (Events Design) — both readable.
- **Entity grid (one line each):** Initiative (strategic banner) -> Project
  (a deliverable, may span both teams) -> Issue (one unit of work). **No**
  sub-issues, blocking, labels, priority, cycles, or saved views.
- **Statuses — collapsed into 4 buckets** (not two full tables):
  - *Not started* — `Triage` / `Backlog` / `Todo`
  - *In progress* — the whole started cluster (incl. review stages)
  - *Done*
  - *Won't ship* — `Canceled` / `Duplicate`
  Enough to interpret "where is this."
- **Triage** = the inbox where new requests land for the team to pick up
  (bridges to the filing flow).

No internal mechanics (handoff invariants, marker-label bounce semantics, `UI`
label, MCP write choreography) appear anywhere in this plugin.

## Filing flow (primary capability) — inline in SKILL.md

Optimized for frequency / low friction:

- **Low-friction capture.** Accept a rough, plain-language description (Russian
  OK). Don't interrogate: draft a good English title (imperative, names the
  outcome, standalone-readable) + an actionable description from what's given.
  **At most one** clarifying question, and only if the issue would otherwise not
  be actionable; otherwise go straight to the draft.
- **Hard defaults, no choices:** `team: "Events Engineering"`, `state: "Triage"`.
  Nothing else. The skill **never** sets labels / priority / project / assignee —
  the PM sets those on the Triage sweep. Stated explicitly in the skill.
- **Quiet dedup before write:** run a lightweight `list_issues query:"…"`;
  surface a candidate only if there's an obvious duplicate, otherwise file
  silently. Keeps Triage clean at almost no friction.
- **Single confirmation:** show the drafted title + description -> one "ok" ->
  `save_issue` create -> return the `ENG-N` link.
- **Translation:** RU -> EN at filing time (the only base invariant that
  propagates here — the tracker is the English canonical record).
- **Batch:** several requests at once -> one separate Triage issue each.

## Read capability — references/reading-the-tracker.md

Loaded on demand. Read-only, spans `ENG` + `DES`. Recipes:

1. **Status of a specific issue** (`ENG-N` / `DES-N`) -> `get_issue`: status (in
   4-bucket terms), assignee, latest update.
2. **"What's happening in Events now"** -> `list_issues` over active statuses
   across `ENG` + `DES` + recent status updates.
3. **Find an issue by topic** -> `list_issues query:"…"`.
4. **Summarize a project** -> `get_project` + its issues + `get_status_updates`.
5. **Exec overview** -> `list_initiatives` -> `list_projects` with status / lead.

All list/get calls scope by team (`team: "Events Engineering"` /
`"Events Design"`), per the MCP rule against workspace-wide pulls.

## Tools (SKILL.md frontmatter)

Reads: `Read`, `mcp__linear-server__list_teams`, `…list_projects`,
`…list_issues`, `…list_initiatives`, `…list_users`, `…list_comments`,
`…list_issue_statuses`, `…get_team`, `…get_project`, `…get_issue`,
`…get_initiative`, `…get_user`, `…get_status_updates`.

Write (exactly one): `mcp__linear-server__save_issue` — create-into-Triage,
constrained by the skill instructions.

**Excluded:** `save_project`, `save_comment`, `save_initiative`,
`save_status_update`, `create_issue_label`; and `list_issue_labels`,
`list_cycles`, `list_milestones` (kept out to stay thin).

## Repo housekeeping

- **`marketplace.json`** — add a plugin entry for `events-tracker-public`.
- **`README.md`** — add a table row; note it is the only plugin for the
  audience outside development and that it does **not** use
  `events-tracker-shared`.
- **`CLAUDE.md`** — update Layout + scope note: `events-tracker-public` reads
  both teams read-only and files only into `ENG` `Triage`. Add a **drift-guard**:
  "the ontology vocabulary is mirrored in `events-tracker-public` — keep in sync
  on entity/status renames."
- **Versioning:** new plugin starts at `0.1.0`. The existing 4 role plugins and
  `events-tracker-shared` are **not** touched (no version bumps).

## Out of scope

- Any write beyond creating a Triage issue (no comments, no edits, no status
  changes, no project/initiative writes).
- Any exposure of internal process/invariants.
- `Artem Personal` or any Linear team other than `ENG` / `DES`.
- Splitting or modifying `events-tracker-shared` or the role plugins.
