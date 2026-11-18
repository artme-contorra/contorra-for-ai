# The scope and the prod gate

Loaded from `releasing` → `SKILL.md` steps 2 to 4. Commands and the workflow names are in
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
- Open, Artem's: what passes QA gave on earlier builds mean for this one when many PRs merged since
  (for the first CI release, the passes given on 2.2.0 / 2.2.1 against 2.2.3). Ask him in one line
  whether QA rechecks the scope on this build or he accepts the earlier passes; do not choose, and
  do not count such a feature green until he answers.
- A release ahead of QA (QA checks after it, as 2.2.5 for a user session that could not wait) is Artem's
  call per release: it goes into `known_issues` as such ("QA after the release, accepted by Artem"), and
  the features still move to `Released` at the apply (`after-apply.md`).

## Part 2: no open blocker in `ENG` `Triage`

- Read `ENG` `Triage` (`list_issues team:"Events Engineering" state:"Triage"`), and the open `2.0`
  issues related to a feature in scope at any status before `Done`.
- A blocker is a defect that breaks a feature in scope for its users, or loses or corrupts data. If
  the host workspace defines blocking more precisely (a triage skill or a decision record), that
  definition wins.
- For each candidate, say whether it is a blocker and why, in one line. Anything still unswept in
  `Triage` that touches a feature in scope is read, not skipped.
- An open blocker is red. It ships only as a known issue Artem accepts.

## Part 3: the prod dry runs

- Read in the plan run, after Artem starts it (`SKILL.md` step 4, § Reading the plan below): Core's
  two dry runs and each Surfaces family's. Green only when every one passed.
- A failed dry run fails the plan's `Summary` whatever `known_issues` says: no known issue covers it.
  The answer is a fix (a precondition, a form, the next build).
- Before the plan runs, this part is "not read yet", not green.

## The nightly: read, not gating

- The nightly (on `main` of Core and of Surfaces, mutation included; `workflows.md`) is a standing measure
  of the tests, not part of the gate. The plan runs none and waits for none; it reads the latest green
  nightly on a descendant of the build and shows it as "read, not gating".
- Report it in one line per repository: the latest nightly's result and commit, and whether that
  commit carries the build (`gh api repos/<repo>/compare/<build sha>...<nightly sha> --jq .status`
  answers `identical` or `ahead`). A red or missing nightly is information for Artem, not a red part
  and not a known issue.

## Known issues

A known defect ships only as a written known issue in the release record: the plan's summary, which
gets it from the plan's `known_issues` input (one line). That input is the only way a known issue
reaches the record, so every accepted exception goes in it, Linear-side ones included (a feature not
`QA passed`, a Triage blocker), not only the part the workflow reads itself (scenarios).
Each carries three things, written so someone outside the room can read them later:

- **What**: the defect, with its `ENG-N`.
- **Why it is not a blocker**: one sentence (the feature still works for its users because ...).
- **Who accepted it**: Artem, or whoever he named.

The agent drafts the wording; Artem accepts or changes it before the plan runs. The plan command then
carries them joined into one line, e.g.
`-f known_issues="ENG-240: <what>, not a blocker because <why>, accepted by Artem; ENG-244: ..."`.
A red part of the gate with no accepted known issue stops the release; on the workflow's side, red
scenarios with an empty `known_issues` fail the plan's `Summary` job, and the apply
refuses a version whose latest completed plan did not pass `Summary`. A newer failed plan overrides an
older green one, so a plan rerun to change the known issues must itself pass. The workflow checks only
that `known_issues` is not empty, not that it names the red part: that every red part is covered, and
covered in words someone can read later, is this skill's check.

The record lives only in the plan run's summary, which goes with the run's retention; the `prod/v` tag
names the run but does not carry the known issues. Whether to keep them elsewhere is Artem's open
question (`workflows.md` § Open): if the release has known issues, ask him in one line where they are
kept beyond the run; do not choose.

## What to show Artem

One block, before any link: the version; the scope as a list of `FEAT-N` and titles; each gate part
(0 to 3) green, red or not read yet with one line why; the nightly line, marked "read, not gating"; the proposed known issues, already in the one line the plan
command will carry. Then the question: go to plan, or not.

## Before the plan

Put these to Artem with the gate block, each in one line, before the plan link:

- **Open questions he has not answered for this release** (`workflows.md` § Open), without a proposed
  answer: whether this release needs the `--long` stand scenarios, which CI never runs; what earlier
  QA passes mean for this build (Part 1).
- **The runners**: what is running or queued on Core's and Surfaces' one runner (`workflows.md`
  § Commands), and whether Core's nightly window (from 20:00 UTC, up to 6 hours) is near. A queue can
  cancel a Core dry run or deploy on its wait; propose a time outside it.
- **The preconditions** (`workflows.md` § Preconditions) only if one is known to be missing.
- **Prod's ground and forms** (`prod-checklist.md` § Before the plan), every time: no workflow reads
  them, and a gap there fails the apply halfway or leaves prod cold for the first user.

## Reading the plan

- **The migrations.** Core's migration job has no dry run, so every migration between the version
  prod runs and this one runs first on prod data during the apply. The version prod runs is the latest
  `prod/v2.*` tag; with none (prod deployed by hand before CI), ask Artem which version prod runs.
  List the Core migrations added between that tag and `v<version>` (`gh api
  repos/maivanov/Contorra-Events-Core/compare/v<prod>...v<version>` and the migration files it touches),
  one line each, and say they are not rehearsed.
- **The console** is not previewed and is rebuilt from source at the tag during the apply: say so.
- **Diffs against the scope**: as `SKILL.md` step 4 says.
