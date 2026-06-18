# CLAUDE.md

Claude Code plugin marketplace for Contorra. Ships the **Events tracker** suite:
`events-tracker-shared` (base) + one role plugin each for PM / designer / dev / QA,
plus `events-tracker-public` — a standalone plugin for stakeholders outside the dev team.
See `README.md` for the plugin map, install, and MCP setup — not repeated here.

There is no build/test/lint toolchain. The "code" is `SKILL.md` files, their
`references/*.md`, and JSON manifests (`plugin.json`, `marketplace.json`, `.mcp.json`).

## Editing rules (must follow)

- **Bump the version on every content change.** Each `plugin.json` `version` is the
  source of truth (SKILL.md files carry no version). Bump it in the *same commit* as
  the change: patch = fix/wording, minor = new reference/section/rule. Unbumped
  changes may not propagate to installed users.
- **Bump the right plugin.** Editing `events-tracker-shared` affects all roles — bump
  shared. Editing a role plugin's own files — bump only that role.
- **Canonical docs win.** The source-of-truth process/ontology docs live outside this
  repo (Events management workspace, `research/processes/`). On any conflict, patch the
  skill to match the doc — never the reverse.
- **MCP must be stdio.** Plugin-bundled `.mcp.json` must use `command`/`args` (see
  `events-tracker-shared/.mcp.json`). An HTTP-type (`"type":"http"`) entry does not
  register on install (verified 2026-06-04); `mcp-remote` bridges stdio → HTTP.
- **Scope is the two Events teams only** (`ENG` Events Engineering, `DES` Events
  Design). `Artem Personal` and any other Linear team are out of scope.
- **`events-tracker-public` is standalone and outsider-facing.** It does NOT
  depend on `events-tracker-shared` and must NOT carry internal process
  (invariants, labels, review/cycle mechanics). It reads both teams read-only
  and writes only by creating issues in `ENG` `Triage`. Its thin ontology
  vocabulary (entity names, status buckets) mirrors `events-tracker-base` — keep
  the two in sync on any entity/status rename. Editing it bumps only its own
  `plugin.json`.

## Layout

- `events-tracker-shared/` — base catalog/ontology/invariants + bundled Linear MCP; a
  dependency of every role, never installed directly.
- `events-tracker-{pm,designer,dev,qa}/` — one role each; each depends on shared.
- `events-tracker-public/` — standalone outsider-facing plugin; bundles its own
  Linear MCP; no dependency on shared. One skill: thin ontology + read recipes +
  Triage filing.
- `<plugin>/skills/<skill>/SKILL.md` — skill entry; `references/*.md` — per-workflow detail.
