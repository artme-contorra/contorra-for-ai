# contorra-for-ai

Claude Code plugin marketplace for AI-assisted work at Contorra. Currently ships the **Events tracker** suite: a shared base plus one plugin per role on the Events team, so each person's Claude works the Linear tracker (teams `ENG` Events Engineering + `DES` Events Design) by the team's conventions — right scope, right guard-rails, right MCP-tool usage.

## Plugins

| Plugin | Consumer | What it does |
|---|---|---|
| `events-tracker-shared` | (dependency) | Base map, ontology, invariants, and Linear MCP-tool guide. Bundles the Linear MCP server. Auto-loaded alongside any role plugin — not installed directly. |
| `events-tracker-pm` | PM / tracker-owner | Create Project pairs (Design + Impl), sweep `ENG` Triage, plan cycles, status reviews. |
| `events-tracker-designer` | Designer | Self-pull Design Issues in `DES`, submit for dev review, respond to bounces, review impl on staging in `Design Review`. |
| `events-tracker-dev` | Web devs | Pick up Impl Issues, walk `In Progress → In Review → In Staging`, hand off to Design Review / QA, respond to bounces, sign off on designs for implementability. |
| `events-tracker-qa` | QA | Pick up Impl Issues in `QA` on staging, approve to `Done` (= release to prod) or reject with a checklist, route out-of-scope findings to Triage. |

Each role plugin depends on `events-tracker-shared`; installing a role pulls the base automatically. Install exactly one role plugin — it pairs with the base to read as one combined context.

## Install

```
/plugin marketplace add artme-contorra/contorra-for-ai
/plugin install events-tracker-dev@contorra-for-ai
```

Swap the last line for your role: `events-tracker-pm`, `events-tracker-designer`, or `events-tracker-qa`. Don't install `events-tracker-shared` directly — it comes in as a dependency.

## Linear MCP

The Linear MCP server is bundled via `events-tracker-shared` (`mcp-remote` → `https://mcp.linear.app/mcp`); no separate setup. On the first tool call you'll be prompted to authorize via Linear OAuth in the browser — the skills then run against live Linear under your own account.

Note for plugin authors: plugin-bundled MCP configs must use stdio transport (`command` / `args`); an HTTP-type (`"type": "http"`) entry in a plugin's `.mcp.json` does not register on install (verified 2026-06-04). `mcp-remote` bridges stdio → HTTP.

## Versioning

Per-plugin semver in each `plugin.json` is the single source of truth (SKILL.md files carry no version). **Every content change bumps the plugin's version in the same commit** — patch for fixes / wording, minor for new references / sections / rules. Unbumped changes may not propagate to installed users. Shared-plugin changes affect all roles — shared bumps itself; role plugins bump only when their own files change.

## Canonical process docs

These plugins are the operating surface, not the source of truth. The canonical process and ontology design live in the Events management workspace (`research/processes/2026-06-04_events-tracker-ontology-design.md`, `2026-06-03_events-delivery-flow-design.md`, `2026-06-04_events-tracker-skills-design.md`). On any conflict, the canonical docs win — patch the skill, not the doc.
