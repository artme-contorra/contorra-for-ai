# The scope and the prod gate

Loaded from `releasing` → `SKILL.md` steps 2 and 3. Commands and the provisional workflow names are in
`../../../references/workflows.md`.

## The scope

- The build's Surfaces commit is in its manifest (the tag `v<version>` and the test-build run's
  summary). Read `release.json` at that commit:
  `gh api repos/maivanov/Contorra-Events-Surfaces/contents/surfaces/web/client/src/shared/flags/release.json?ref=<sha> --jq .content | base64 --decode`.
- The scope is the features opened there that are not yet `Released` in `FEAT` (`get_issue` on each,
  or `list_issues team:"Features"` and match). A feature already `Released` by an earlier version is
  carried, not released again.
- Cross-check with the manifest's merged PRs: a PR that names a `FEAT-N` not in `release.json` touches
  a closed feature; say so, it is not in scope.

## Part 1: every feature in scope `QA passed`

- List each feature in scope with its status. Green only when every one is `QA passed`.
- `Ready for QA`, `QA failed`, `In development`: red. Name each one and what QA last wrote on it
  (`list_comments`).
- QA accepts a feature on the test stand. Check that QA's pass was on this build or one whose change to
  that feature this build does not alter; if QA passed it on an older build and this build changed it,
  say so (that is a judgement for Artem, not a green).

## Part 2: no open blocker in `ENG` `Triage`

- Read `ENG` `Triage` (`list_issues team:"Events Engineering" state:"Triage"`), and the open `2.0`
  issues related to a feature in scope at any status before `Done`.
- A blocker is a defect that breaks a feature in scope for its users, or loses or corrupts data. If
  the host workspace defines blocking more precisely (a triage skill or a decision record), that
  definition wins.
- For each candidate, say whether it is a blocker and why, in one line. Anything still unswept in
  `Triage` that touches a feature in scope is read, not skipped.
- An open blocker is red. It ships only as a known issue Artem accepts.

## Part 3: a green nightly

- The nightly runs on `main` of Core and of Surfaces. For each repository, find the latest green
  nightly run and its commit, and check that commit is the build's commit or a descendant of it:
  `gh api repos/<repo>/compare/<build sha>...<nightly sha> --jq .status` answers `identical` or `ahead`.
- No such run is not red: the plan runs the nightly suite on the build's commits first. Say that the
  plan will take longer.
- A red nightly on a descendant is red unless the failure is shown to be in a commit after the build.

## Known issues

A known defect ships only as a written known issue in the release record (the plan's summary). Each
carries three things, written so someone outside the room can read them later:

- **What**: the defect, with its `ENG-N`.
- **Why it is not a blocker**: one sentence (the feature still works for its users because ...).
- **Who accepted it**: Artem, or whoever he named.

The agent drafts the wording; Artem accepts or changes it before the plan runs. A red part of the gate
with no accepted known issue stops the release.

## What to show Artem

One block, before any link: the version; the scope as a list of `FEAT-N` and titles; each gate part
green or red with one line why; the proposed known issues. Then the question: go to plan, or not.
