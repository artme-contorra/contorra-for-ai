# Cutting a test build: the procedure

Loaded from `cutting-a-test-build` → `SKILL.md`. Workflow names, inputs and commands are in
`../../../references/workflows.md`.

## 1. What merged since the last build

- The last build is the highest `v2.<stage>.<build>` tag on Surfaces (Core carries the same tag).
  Read the tags with `gh api` (workflows.md § Commands).
- List the PRs merged into `main` since that tag in Core and in Surfaces:
  `gh pr list --repo <repo> --base main --state merged --search "merged:>=<tag date>" --json number,title,mergedAt,url`,
  cross-checked against `git log <tag>..origin/main` in a fresh clone or `gh api repos/<repo>/compare/<tag>...main`.
- For each PR, the `ENG-N` it names (branch, title or description) and that issue's state
  (`get_issue`). On 2.0 a merged issue should be in `QA`; one that is not is a finding for the
  issue's owner (the PR did not name it, or the integration is not connected for that repository).
- Proto: list what merged there too, but Proto is carried by the pin, not built.

Say what the build is for. Typical answers: "the fixes for ENG-231 and ENG-235 that QA is waiting on";
"FEAT-12 opens (release.json changed in #230)"; "a week of small fixes, nothing QA is waiting on, can
wait". The last one is a reason not to build.

## 2. What would stop the workflow

- **The Proto pin.** Core and Surfaces must pin the same Proto commit at their `main`. Read each
  `proto` submodule entry: `gh api repos/<repo>/contents/proto?ref=main --jq .sha`. Different pins: the
  side that is behind needs its pin PR merged first; say which.
- **`floor` on `main`.** CI runs on pull requests, not on `main`, so `main`'s merge commit carries no
  `floor` of its own; the workflow reads it on that commit or its second parent, the head of the PR
  that made it. Read the same: `gh api repos/<repo>/commits/main --jq '.sha, .parents[1].sha'`, then
  `gh api repos/<repo>/commits/<PR head sha>/check-runs --jq '.check_runs[] | select(.name=="floor") | .conclusion'`
  (the latest run counts). Red, or missing on both the commit and its second parent (a squash merge or
  a direct push), is the workflow's stop too: say which repository and stop.
- **Something to build.** If neither Core's nor Surfaces' `main` moved since the last tag, the
  workflow refuses; § 1 already shows that.
- **The stage.** The default is the current stage. A new stage is Artem's call; ask, never assume.

## 3. Dispatch and wait

- **Spent versions first.** Read the previous test-build run (`gh run list`, then
  `gh run view <id> --json conclusion,jobs`). If it ended untagged (its `tag the pair and record the
  manifest` job failed, or never ran, and no `v<version>` tag exists for it) after any family's build
  or deploy job ran, its version may be spent: its images may already be pushed. That run's "Not
  tagged" note names the `skipped` value to pass; it is on the run page only, so ask Artem to read it
  out, or derive it: the run's version was one past the highest `v2.<stage>.*` tag, or one past the
  highest `skipped` it was given. Pass every spent version since the last tag, accumulated and
  space-separated (`-f skipped="2.2.4 2.2.5"`). A version not named that way makes the next build's
  tagging refuse the gap; a version named that was never spent wastes a number.
- Dispatch the test-build workflow from `main` of Surfaces (workflows.md § Commands). Pass `stage`
  only when Artem named a new one, `skipped` only as above.
- Find the run it started (`gh run list`, the newest `workflow_dispatch` run created after the
  dispatch) and give its URL.
- Watch it to the end (`gh run watch --exit-status`). The run deploys to the shared test stand; tell
  the person that test is changing while it runs, since QA may be on it.
- Do not start a second test build while one is running.

## 4. Reading the result

- **Green.** Read the manifest from the tag `v<version>` (workflows.md § Outputs; the run's summary
  is not readable with `gh run view`): version, commits, Proto pin, images, `behind:`, `scenarios:`,
  merged PRs and their `ENG` issues. Check the version tag exists on both Core and Surfaces.
- **Red.** `gh run view <id> --json conclusion,jobs`, the run's `::error::` annotations, then
  `--log-failed`. Name the job by its role (workflows.md § Jobs): compose (the reads of § 2 found
  late, the numbering, or "nothing moved"), Core's deploy (a separate Core run: follow its URL in the
  job's log), a Surfaces family's build and deploy, the scenarios, or the tagging. Give the first
  failing line in plain words. A deploy failure leaves test on the previous build or partly updated:
  say which families moved. A failure before the tagging leaves the version untagged and maybe spent:
  say so, and carry it into the next dispatch (§ 3).
- **Tagged, scenarios red.** The run is red but the build is tagged and on test: the tagging job
  stops after the tag with "deployed and tagged, but the scenarios on test are ...". The build is not
  fit for QA and cannot be released: say so and do not draft a QA post for it.
- Never re-run a red run blindly. On a test-build run only: re-run a job that failed on the machine
  (network, a runner restart), once, and say so. A red check on the code needs a fix PR and a new
  build. Plan, apply and prod runs are never re-run by the agent (workflows.md § Commands).

## 5. What the build does not do

- It moves no `ENG` status: the integration already moved merged 2.0 issues to `QA`, and QA moves them
  on after checking.
- It moves no `FEAT` status. A feature whose fix this build carries goes `QA failed` → `Ready for QA`
  at "the sync when a fix reaches the stand" (`events-tracker-base` § Statuses → `FEAT`), which is not
  this skill. List such features in the QA post draft so Artem sees them; do not move them.
- It does not touch prod and does not make the build a release. A release is a later step, on a build
  QA has checked (`releasing`).
