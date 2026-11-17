# After a green apply

Loaded from `releasing` → `SKILL.md` step 6. Only after the apply run is green: every family deployed,
the host checks passed, the release tag `prod/v<version>` pushed on Core and Surfaces (`v<version>`
alone is the test build).

## 1. `FEAT` features to `Released`

- The list is the scope from `gate.md`: the features this version carries that were not `Released`.
- Show Artem the list (`FEAT-N` and title) and ask once for a yes to the whole list. Then, per feature:

```
mcp__linear-server__save_issue
  id: "FEAT-N"
  state: "Released"
```

- `Released` is the only write. No label, no mark, no `iteration` label, no description edit
  (`events-tracker-base` Invariant 6: marks are never set by hand). No comment unless Artem asks.
- No `ENG` status moves. 2.0 issues reached `Done` when QA checked them on test; a release does not
  touch them.
- Echo the result: which features moved, any that failed to move and why.

## 2. The team post draft

Where it goes:

- `#team-events-ru`, in Russian, for every release.
- `#team-events`, in English, only when the release is for people outside the team (Artem says so).
  When in doubt, draft only the first and ask.

What it says, in this order:

1. The version is on prod, and where (the public hosts by name).
2. What it brings, as features users can now use: one line per feature, in plain words, with `FEAT-N`
   in the Russian post only.
3. Known issues from the release record, one line each, without internal reasoning.
4. Anything people need to do (sign in again, a new address), only if there is something.

Leave out: images, commits, workflow runs, gate details.

Hand the draft to Artem in chat; he edits and sends it. If the host workspace has a skill for writing
in Artem's voice, follow it. No long dashes.
