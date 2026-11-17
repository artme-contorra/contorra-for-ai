# The scope and the prod gate

Loaded from `releasing` → `SKILL.md` steps 2 and 3. Commands and the workflow names are in
`../../../references/workflows.md`.

## The scope

- The build's Surfaces commit is in its manifest, the annotated tag `v<version>` (read it as
  `workflows.md` § Outputs says; the test-build run's summary is not readable with `gh run view`).
  Read `release.json` at that commit:
  `gh api repos/maivanov/Contorra-Events-Surfaces/contents/surfaces/web/client/src/shared/flags/release.json?ref=<sha> --jq .content | base64 --decode`.
- The scope is the features opened there that are not yet `Released` in `FEAT` (`get_issue` on each,
  or `list_issues team:"Features"` and match). A feature already `Released` by an earlier version is
  carried, not released again.
- Cross-check with the manifest's merged PRs: a PR that names a `FEAT-N` not in `release.json` touches
  a closed feature; say so, it is not in scope.

## Part 0: the build's scenarios on test

- The tag's manifest has `scenarios: <result>`. Green only when it is `success`.
- A test build is tagged even when its scenarios fail, so the latest `v2.*` tag can be a build that is
  on test but not fit for release. The plan reads the same line and goes red on anything but
  `success`.
- Red: name the result and the test-build run (`run:` in the manifest). It ships only as a known issue
  Artem accepts; usually the answer is the next build.

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

- The nightly runs on `main` of Core and of Surfaces (`nightly.yml`). For each repository, find the latest green
  nightly run and its commit, and check that commit is the build's commit or a descendant of it:
  `gh api repos/<repo>/compare/<build sha>...<nightly sha> --jq .status` answers `identical` or `ahead`.
- No such run is not red: the plan then runs the nightly on `main`, a descendant of the build (not on
  the build's own commits), before it summarises. Say that the plan will take longer, up to several
  hours.
- A red nightly on a descendant is red unless the failure is shown to be in a commit after the build.
  A red nightly the plan ran itself is judged the same way: show whether the failing commit is after
  the build. Today a known issue cannot carry a red nightly (Surfaces #226 at `370139548`): the plan
  counts only a green nightly, runs one on `main` when there is none, and a red nightly it ran fails
  the plan run whatever `known_issues` says, so the apply refuses the version. A red nightly needs a
  fix, a green nightly and a new plan; say so before handing Artem the plan.

## Known issues

A known defect ships only as a written known issue in the release record: the plan's summary, which
gets it from the plan's `known_issues` input (one line). That input is the only way a known issue
reaches the record, so every accepted exception goes in it, Linear-side ones included (a feature not
`QA passed`, a Triage blocker), not only the parts the workflow reads itself (scenarios, nightlies).
Each carries three things, written so someone outside the room can read them later:

- **What**: the defect, with its `ENG-N`.
- **Why it is not a blocker**: one sentence (the feature still works for its users because ...).
- **Who accepted it**: Artem, or whoever he named.

The agent drafts the wording; Artem accepts or changes it before the plan runs. The plan command then
carries them joined into one line, e.g.
`-f known_issues="ENG-240: <what>, not a blocker because <why>, accepted by Artem; ENG-244: ..."`.
A red part of the gate with no accepted known issue stops the release; on the workflow's side, red
scenarios with an empty `known_issues` fail the plan (a red nightly fails it either way, Part 3), and
the apply refuses a version with no successful plan.

## What to show Artem

One block, before any link: the version; the scope as a list of `FEAT-N` and titles; each gate part
(0 to 3) green or red with one line why; the proposed known issues, already in the one line the plan
command will carry. Then the question: go to plan, or not.
