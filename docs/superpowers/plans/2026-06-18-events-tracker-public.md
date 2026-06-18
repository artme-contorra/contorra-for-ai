# events-tracker-public Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a standalone Claude Code plugin `events-tracker-public` that lets stakeholders outside the dev team read the Events tracker (ENG+DES, read-only) and file requests into ENG Triage — with no exposure to internal team process.

**Architecture:** A self-contained plugin with no dependency on `events-tracker-shared`. It bundles its own Linear MCP (`.mcp.json`, identical to shared's stdio config) and ships one skill (`events-tracker-public`) whose SKILL.md leads with the filing flow (the primary, most-frequent path, inline) plus a deliberately thin curated ontology, and a single on-demand reference for read recipes. Existing role plugins and `events-tracker-shared` are untouched.

**Tech Stack:** Claude Code plugin format — JSON manifests (`plugin.json`, `.mcp.json`, `marketplace.json`) + Markdown skills (`SKILL.md`, `references/*.md`). No build/test/lint toolchain; verification = JSON parse validation + structural inspection. `mcp-remote` bridges stdio → Linear's HTTP MCP. Linear MCP tools are the runtime surface.

**Reference spec:** `docs/superpowers/specs/2026-06-18-events-tracker-public-plugin-design.md`

---

## File structure

Files created (all new — nothing in the existing suite is modified except shared repo metadata):

- `events-tracker-public/.claude-plugin/plugin.json` — plugin manifest, v0.1.0, no dependencies.
- `events-tracker-public/.mcp.json` — bundled Linear MCP (verbatim copy of shared's).
- `events-tracker-public/skills/events-tracker-public/SKILL.md` — the skill: thin ontology + inline filing flow + scope/guard-rails + pointer to reading reference.
- `events-tracker-public/skills/events-tracker-public/references/reading-the-tracker.md` — read recipes, loaded on demand.

Files modified (repo metadata only):

- `marketplace.json` — add plugin entry.
- `README.md` — add table row + note.
- `CLAUDE.md` — Layout + scope note + drift-guard.

Work happens on branch `events-tracker-public` (already created; spec already committed there).

---

### Task 1: Plugin manifest + bundled MCP

**Files:**
- Create: `events-tracker-public/.claude-plugin/plugin.json`
- Create: `events-tracker-public/.mcp.json`

- [ ] **Step 1: Create the plugin manifest**

Create `events-tracker-public/.claude-plugin/plugin.json`:

```json
{
  "name": "events-tracker-public",
  "version": "0.1.0",
  "description": "Read-only Events tracker access (Linear ENG + DES) plus filing requests into ENG Triage, for stakeholders outside the dev team. Self-contained; bundles the Linear MCP server; no internal team process.",
  "author": {
    "name": "Artem Kononov",
    "email": "ak@contorra.com"
  },
  "keywords": [
    "linear",
    "events",
    "tracker",
    "stakeholder",
    "public"
  ]
}
```

Note: no `dependencies` key — this plugin is standalone and must NOT depend on `events-tracker-shared`.

- [ ] **Step 2: Create the bundled MCP config**

Create `events-tracker-public/.mcp.json` (verbatim copy of `events-tracker-shared/.mcp.json` — stdio transport per the repo's MCP rule):

```json
{
  "mcpServers": {
    "linear-server": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://mcp.linear.app/mcp"]
    }
  }
}
```

- [ ] **Step 3: Verify both JSON files parse**

Run:
```bash
python3 -m json.tool events-tracker-public/.claude-plugin/plugin.json > /dev/null && \
python3 -m json.tool events-tracker-public/.mcp.json > /dev/null && echo "JSON OK"
```
Expected: `JSON OK`

- [ ] **Step 4: Verify the MCP config matches shared exactly**

Run:
```bash
diff <(python3 -m json.tool events-tracker-public/.mcp.json) <(python3 -m json.tool events-tracker-shared/.mcp.json) && echo "MCP MATCHES"
```
Expected: no diff output, then `MCP MATCHES`.

- [ ] **Step 5: Commit**

```bash
git add events-tracker-public/.claude-plugin/plugin.json events-tracker-public/.mcp.json
git commit -m "Add events-tracker-public plugin manifest and bundled MCP"
```

---

### Task 2: SKILL.md (thin ontology + inline filing flow)

**Files:**
- Create: `events-tracker-public/skills/events-tracker-public/SKILL.md`

- [ ] **Step 1: Write the skill**

Create `events-tracker-public/skills/events-tracker-public/SKILL.md` with exactly this content:

````markdown
---
name: events-tracker-public
tools:
  - Read
  - mcp__linear-server__list_teams
  - mcp__linear-server__list_projects
  - mcp__linear-server__list_issues
  - mcp__linear-server__list_initiatives
  - mcp__linear-server__list_users
  - mcp__linear-server__list_comments
  - mcp__linear-server__list_issue_statuses
  - mcp__linear-server__get_team
  - mcp__linear-server__get_project
  - mcp__linear-server__get_issue
  - mcp__linear-server__get_initiative
  - mcp__linear-server__get_user
  - mcp__linear-server__get_status_updates
  - mcp__linear-server__save_issue
description: >
  Stakeholder-facing access to the Events tracker in Linear (teams `ENG`
  Events Engineering and `DES` Events Design) for people outside the dev team —
  leadership, product owners, the CEO, the PM director. Two capabilities:
  (1) FILE a request — turn a plain-language description (Russian OK) into a
  well-formed English issue landed in `ENG` `Triage` for the team to pick up;
  this is the primary, most-frequent use. (2) READ the tracker, read-only,
  across the whole Events product (`ENG` + `DES`): status of an issue, what's
  happening now, find an issue by topic, summarize a project, exec overview of
  initiatives and projects. Self-contained — exposes none of the team's
  internal workflow. Strictly scoped to the two Events teams; `Artem Personal`
  and any other Linear team are out of scope. Use whenever someone outside the
  dev team wants to check on Events work or submit a request / feature idea /
  feedback, even when Linear isn't named explicitly. Trigger phrases:
  "file a request", "feature request", "submit feedback", "what's happening in
  Events", "status of ENG-N", "how's the Events product", "зафайлить реквест",
  "что в продукте events", "статус задачи events".
---

Stakeholder-facing skill for the Events tracker, for people **outside** the dev
team. It does exactly two things and nothing else: **file a request into `ENG`
`Triage`** (the primary path), and **read the tracker** read-only across `ENG`
and `DES`. It deliberately exposes none of the team's internal workflow — no
labels, no review choreography, no cycle mechanics. The team's own roles (PM /
dev / designer / QA) run separate plugins; this one never does their work.

## What you're looking at

A minimal map — just enough to read and to file. Nothing more is needed here.

- **Teams:** `ENG` (Events Engineering) and `DES` (Events Design). Both are
  readable. New requests are filed only into `ENG`.
- **Entity grid:**
  - **Initiative** — a strategic banner several projects sit under.
  - **Project** — a concrete deliverable; may span both teams (design + build).
  - **Issue** — one unit of work, with one assignee. Identifiers `ENG-N` / `DES-N`.
- **Status, in four buckets** (the team uses finer-grained statuses internally;
  for reading, collapse them):
  - **Not started** — `Triage`, `Backlog`, `Todo`.
  - **In progress** — anything actively being worked or reviewed.
  - **Done** — shipped.
  - **Won't ship** — `Canceled`, `Duplicate`.
- **Triage** — the inbox on `ENG` where new requests land. The team triages from
  there. Filing a request (below) means landing a clean issue in `Triage`.

## Filing a request into `ENG` `Triage` (primary)

Turn whatever the person describes — a feature idea, a bug they hit, feedback
after using the platform — into one well-formed issue in `Triage`. Optimized for
low friction: don't interrogate, draft and confirm.

**Flow:**

1. **Take the description as-is** (Russian is fine — you'll translate).
2. **Draft the issue yourself:**
   - **Title** — a clear English imperative that names the outcome, standalone-
     readable (e.g. `Add CSV export to the attendee list`, `Fix broken date
     filter on the sessions page`). No internal prefixes/suffixes.
   - **Description** — a short English paragraph with enough context to be
     actionable: what they want or what went wrong, and where/why. Translate
     from Russian if needed — the tracker is the English-language record.
   - Ask **at most one** clarifying question, and only if the request is too
     vague to be actionable without it. Otherwise go straight to the draft.
3. **Quiet duplicate check:** before writing, run
   `list_issues team:"Events Engineering" query:"<key words from the request>"`.
   Surface a match **only** if there's an obvious duplicate ("This looks like it
   may already be tracked as ENG-N — file anyway?"). Otherwise proceed silently.
4. **Show the draft and get one confirmation** — display the title + description
   and ask the person to confirm before anything is written.
5. **Create it** with `save_issue`:
   - `team: "Events Engineering"`
   - `title`, `description` (the drafted English)
   - `state: "Triage"`
   - **Nothing else.** Do **not** set `labels`, `priority`, `project`, or
     `assignee` — the team sets those when they triage. Setting them here is
     wrong.
6. **Return the `ENG-N` link** so they can follow it.

**Batch:** if several requests come at once, file each as its own `Triage`
issue.

This `save_issue` create is the **only** write this skill performs. It never
edits issues, changes statuses, comments, or writes projects / initiatives.

## Reading the tracker

Read-only, across `ENG` + `DES`. Report statuses in the four-bucket language
above — don't expose the team's internal status names unless asked. For the
recipes (status of an issue, what's happening now, find by topic, project
summary, exec overview), load `references/reading-the-tracker.md`.

Always scope list/get calls by team (`team: "Events Engineering"` or
`team: "Events Design"`) so queries don't pull workspace-wide.

## Scope and guard-rails

- **Read freely.** All `list_*` / `get_*` calls are unrestricted.
- **One write only:** creating an issue in `ENG` `Triage`, after showing the
  draft and getting confirmation. No other writes exist for this skill.
- **Scope is `ENG` + `DES` only.** `Artem Personal` and every other Linear team
  are out of scope.
- **No internal process.** This skill carries no team workflow, invariants, or
  label/cycle mechanics — and shouldn't grow them. If a request needs that
  depth, it belongs to a role plugin, not here.
````

- [ ] **Step 2: Verify the YAML frontmatter parses and tool list is correct**

Run:
```bash
python3 -c "
import re, sys
text = open('events-tracker-public/skills/events-tracker-public/SKILL.md').read()
assert text.startswith('---\n'), 'no frontmatter'
fm = text.split('---\n', 2)[1]
try:
    import yaml
    data = yaml.safe_load(fm)
    tools = data['tools']
except ImportError:
    tools = re.findall(r'^\s*-\s*(\S+)', fm, re.M)
write_tools = [t for t in tools if any(t.endswith(w) for w in ['save_project','save_comment','save_initiative','save_status_update','create_issue_label'])]
assert not write_tools, f'forbidden write tools present: {write_tools}'
assert any(t.endswith('save_issue') for t in tools), 'save_issue missing'
assert any(t.endswith('get_status_updates') for t in tools), 'get_status_updates missing'
print('FRONTMATTER OK -', len(tools), 'tools, save_issue is the only save_*')
"
```
Expected: `FRONTMATTER OK - 15 tools, save_issue is the only save_*` (Read + 14 MCP tools)

- [ ] **Step 3: Verify the skill body has no internal-process leakage**

Run (these internal terms must NOT appear in the public skill body):
```bash
! grep -nE 'marker label|Design approved|QA approved|UI label|blocks|sub-issue|cycle planning|handoff|Design Review|In Staging' events-tracker-public/skills/events-tracker-public/SKILL.md && echo "NO INTERNAL LEAKAGE"
```
Expected: `NO INTERNAL LEAKAGE`

- [ ] **Step 4: Commit**

```bash
git add events-tracker-public/skills/events-tracker-public/SKILL.md
git commit -m "Add events-tracker-public SKILL: thin ontology + inline Triage filing"
```

---

### Task 3: Reading reference

**Files:**
- Create: `events-tracker-public/skills/events-tracker-public/references/reading-the-tracker.md`

- [ ] **Step 1: Write the reference**

Create `events-tracker-public/skills/events-tracker-public/references/reading-the-tracker.md` with exactly this content:

````markdown
# Reading the Events tracker

Read-only recipes over `ENG` (Events Engineering) and `DES` (Events Design).
Always pass `team:` so queries don't pull workspace-wide. Report status in the
four buckets from SKILL.md (Not started / In progress / Done / Won't ship) — not
the team's internal status names — unless the person asks for detail.

## 1. Status of a specific issue

Given an identifier like `ENG-42` or `DES-7`:

```
get_issue id:"ENG-42"
```

Report: the title, which bucket it's in, who it's assigned to, and the most
recent update (pull recent comments with `list_comments issueId:"ENG-42"` if a
fuller picture helps).

## 2. What's happening in Events right now

Active work across both teams:

```
list_issues team:"Events Engineering" state:"started"
list_issues team:"Events Design" state:"started"
```

`state:"started"` covers everything in the **In progress** bucket. Summarize by
project where possible. For a narrative of recent movement, add project status
updates (recipe 4) for the active projects.

## 3. Find an issue by topic

```
list_issues team:"Events Engineering" query:"<topic words>"
list_issues team:"Events Design" query:"<topic words>"
```

`query` searches title + description. Widen or narrow the words if the first
pass misses.

## 4. Summarize a project

```
list_projects team:"Events Engineering"        # find the project by name
get_project id:"<project id or name>"
list_issues project:"<project name>"           # its issues, group by bucket
get_status_updates projectId:"<project id>"     # latest narrative updates
```

Give: what the project delivers (its description), how its issues split across
the four buckets, the lead, and the latest status update.

## 5. Exec overview (initiatives → projects)

```
list_initiatives                                # strategic banners
list_projects team:"Events Engineering"
list_projects team:"Events Design"
```

Roll up: each initiative, the projects under it, each project's coarse status
and lead. This is the "state of the product" view.
````

- [ ] **Step 2: Verify the reference parses as text and references only allowed read tools**

Run (the reference must not instruct any write tool):
```bash
! grep -nE 'save_issue|save_project|save_comment|save_initiative|create_issue_label' events-tracker-public/skills/events-tracker-public/references/reading-the-tracker.md && echo "READ-ONLY REFERENCE OK"
```
Expected: `READ-ONLY REFERENCE OK`

- [ ] **Step 3: Commit**

```bash
git add events-tracker-public/skills/events-tracker-public/references/reading-the-tracker.md
git commit -m "Add events-tracker-public reading recipes reference"
```

---

### Task 4: Register in marketplace

**Files:**
- Modify: `marketplace.json`

- [ ] **Step 1: Add the plugin entry**

In `marketplace.json`, append this object to the end of the `plugins` array (after the `events-tracker-qa` entry — add a comma after the `qa` entry's closing `}`):

```json
    {
      "name": "events-tracker-public",
      "source": "./events-tracker-public",
      "description": "Stakeholder-facing Events tracker access for people outside the dev team — read ENG + DES read-only, file requests into ENG Triage. Standalone; does not use events-tracker-shared."
    }
```

- [ ] **Step 2: Verify marketplace.json parses and lists the new plugin**

Run:
```bash
python3 -c "
import json
d = json.load(open('marketplace.json'))
names = [p['name'] for p in d['plugins']]
assert 'events-tracker-public' in names, names
print('MARKETPLACE OK -', names)
"
```
Expected: `MARKETPLACE OK - ['events-tracker-shared', 'events-tracker-pm', 'events-tracker-designer', 'events-tracker-dev', 'events-tracker-qa', 'events-tracker-public']`

- [ ] **Step 3: Commit**

```bash
git add marketplace.json
git commit -m "Register events-tracker-public in marketplace"
```

---

### Task 5: README row + note

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add the table row**

In `README.md`, in the Plugins table (the `| Plugin | Consumer | What it does |` table), add this row immediately after the `events-tracker-qa` row:

```markdown
| `events-tracker-public` | Leadership / stakeholders (outside dev) | Read ENG + DES read-only (issue status, what's in flight, find by topic, project & initiative summaries); file requests into ENG Triage. Standalone — does not use `events-tracker-shared`. |
```

- [ ] **Step 2: Add the install note**

In `README.md`, immediately after the paragraph that ends "...Install exactly one role plugin — it pairs with the base to read as one combined context." add this paragraph:

```markdown
`events-tracker-public` is the exception: it is for people **outside** the dev team (leadership, product owners) and is fully standalone — it bundles its own Linear MCP, does **not** depend on `events-tracker-shared`, and exposes none of the team's internal workflow (read-only across both teams, plus filing into ENG Triage). Install it on its own.
```

- [ ] **Step 3: Verify both additions are present**

Run:
```bash
grep -q 'events-tracker-public.*Leadership / stakeholders' README.md && \
grep -q 'is the exception' README.md && echo "README OK"
```
Expected: `README OK`

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "Document events-tracker-public in README"
```

---

### Task 6: CLAUDE.md — layout, scope, drift-guard

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update the intro line**

In `CLAUDE.md`, replace this line:

```markdown
Ships the **Events tracker** suite:
`events-tracker-shared` (base) + one role plugin each for PM / designer / dev / QA.
```

with:

```markdown
Ships the **Events tracker** suite:
`events-tracker-shared` (base) + one role plugin each for PM / designer / dev / QA,
plus `events-tracker-public` — a standalone plugin for stakeholders outside the dev team.
```

- [ ] **Step 2: Update the scope rule**

In `CLAUDE.md`, in the "Editing rules" section, replace this bullet:

```markdown
- **Scope is the two Events teams only** (`ENG` Events Engineering, `DES` Events
  Design). `Artem Personal` and any other Linear team are out of scope.
```

with:

```markdown
- **Scope is the two Events teams only** (`ENG` Events Engineering, `DES` Events
  Design). `Artem Personal` and any other Linear team are out of scope.
- **`events-tracker-public` is standalone and outsider-facing.** It does NOT
  depend on `events-tracker-shared` and must NOT carry internal process
  (invariants, labels, review/cycle mechanics). It reads both teams read-only
  and writes only by creating issues in `ENG` `Triage`. Its thin ontology
  vocabulary (entity names, status buckets) mirrors `events-tracker-base` — keep
  the two in sync on any entity/status rename. Editing it bumps only its own
  `plugin.json`.
```

- [ ] **Step 3: Update the Layout section**

In `CLAUDE.md`, in the "Layout" section, add this bullet immediately after the `events-tracker-{pm,designer,dev,qa}/` bullet:

```markdown
- `events-tracker-public/` — standalone outsider-facing plugin; bundles its own
  Linear MCP; no dependency on shared. One skill: thin ontology + read recipes +
  Triage filing.
```

- [ ] **Step 4: Verify the edits landed**

Run:
```bash
grep -q 'events-tracker-public — a standalone plugin' CLAUDE.md && \
grep -q 'outsider-facing' CLAUDE.md && \
grep -q 'mirrors `events-tracker-base`' CLAUDE.md && echo "CLAUDE.md OK"
```
Expected: `CLAUDE.md OK`

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md
git commit -m "Document events-tracker-public in CLAUDE.md (layout, scope, drift-guard)"
```

---

### Task 7: Final validation

**Files:** none (verification only)

- [ ] **Step 1: All JSON manifests parse**

Run:
```bash
for f in marketplace.json events-tracker-public/.claude-plugin/plugin.json events-tracker-public/.mcp.json; do
  python3 -m json.tool "$f" > /dev/null && echo "OK $f" || echo "FAIL $f"
done
```
Expected: `OK` for all three.

- [ ] **Step 2: Plugin directory is complete and self-contained**

Run:
```bash
test -f events-tracker-public/.claude-plugin/plugin.json && \
test -f events-tracker-public/.mcp.json && \
test -f events-tracker-public/skills/events-tracker-public/SKILL.md && \
test -f events-tracker-public/skills/events-tracker-public/references/reading-the-tracker.md && \
! grep -q 'dependencies' events-tracker-public/.claude-plugin/plugin.json && \
echo "PLUGIN COMPLETE & STANDALONE"
```
Expected: `PLUGIN COMPLETE & STANDALONE`

- [ ] **Step 3: Existing suite untouched**

Run (no files outside `events-tracker-public/` and the three metadata files should have changed on this branch beyond the spec/plan docs):
```bash
git diff --name-only main...HEAD | grep -vE '^(events-tracker-public/|marketplace.json|README.md|CLAUDE.md|docs/superpowers/)' && echo "UNEXPECTED CHANGES ABOVE" || echo "EXISTING SUITE UNTOUCHED"
```
Expected: `EXISTING SUITE UNTOUCHED`

- [ ] **Step 4: Final summary commit (if any uncommitted changes remain)**

```bash
git status --porcelain
```
Expected: empty (everything already committed task-by-task). If anything remains, stage and commit it with a descriptive message.

---

## Manual verification (human, post-implementation)

These require a live Linear workspace and Claude Code install — do them by hand after the plan completes:

1. `/plugin marketplace add` the repo, then `/plugin install events-tracker-public@contorra-for-ai`. Confirm it installs **without** pulling `events-tracker-shared`.
2. On first Linear MCP tool call, confirm the OAuth prompt appears and reads work.
3. Ask Claude "what's happening in Events right now" → confirm a read summary across ENG+DES.
4. Give a plain-language (Russian) feature request → confirm Claude drafts an English title+description, does a dup check, asks for one confirmation, and creates an issue in `ENG` `Triage` with no labels/priority/project/assignee set.
