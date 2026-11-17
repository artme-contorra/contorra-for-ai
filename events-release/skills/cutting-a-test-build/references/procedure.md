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

- **After a failed build: retry, or skip.** Read the previous test-build run (`gh run list`, then
  `gh run view <id> --json conclusion,jobs`). If it ended untagged (its `Tag & manifest` job failed
  or never ran, and no `v<version>` tag exists for it), decide by whether a `main` moved since:
  - **Neither Core's nor Surfaces' `main` moved:** retry with the same inputs (the same `stage`, the
    same `skipped`, or none). The build takes the same number again and deploys the images already
    pushed as that version, without rebuilding them. Core's build gets the same rule in Core#64; until
    that is on Core's `main`, a retry fails in `Core → test` whenever the failed run had pushed Core's
    images: say so, and the build waits for a new commit on Core's `main`.
  - **A `main` moved, and the failed run pushed an image of its version:** that version is spent. Pass
    it, with every version spent before it since the last tag, as `skipped`, space-separated
    (`-f skipped="2.2.4 2.2.5"`). A repository that did not move and had pushed an image of the spent
    version cannot build until its `main` gets a new commit: say which.
  - The run's "Not tagged" note says the same and names the value; it is on the run page only, so ask
    Artem to read it out, or derive the version: one past the highest `v2.<stage>.*` tag, or one past
    the highest `skipped` the run was given. Whether an image was pushed: the build job of that family
    succeeded (Core: `Core → test`'s Core run; Surfaces: `Surfaces → test / <family>`).
  - Compose checks `skipped` before anything deploys: a version that is not a gap from the next
    untagged number on stops the run with nothing deployed.
- Dispatch the test-build workflow from `main` of Surfaces (workflows.md § Commands). Pass `stage`
  only when Artem named a new one, `skipped` only as above. The run is titled
  `Test build · current stage` (or `· stage <n>`).
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
  `--log-failed`. Name the job (workflows.md § Runs and jobs): `Compose` (the reads of § 2 found
  late, the numbering, a `skipped` that is no gap, or "nothing moved"), `Core → test` (a separate Core
  run, `Deploy all → test · <version>`: follow its URL in the job's log), `Surfaces → test / <family>`,
  the scenarios, or `Tag & manifest`. Give the first
  failing line in plain words. A deploy failure leaves test on the previous build or partly updated:
  say which families moved. A failure before the tagging leaves the version untagged: say so, and whether
  the next dispatch is a retry with the same inputs or carries it as `skipped` (§ 3).
- **Tagged, scenarios red.** The run is red but the build is tagged and on test: the tagging job
  stops after the tag with "deployed and tagged, but the scenarios on test are ...". The build is not
  fit for QA and cannot be released: say so and do not draft a QA post for it.
- Never retry a red run blindly. On a test build only: when a job failed on the machine (network, a
  runner restart), retry once by dispatching again with the same inputs (§ 3), and say so. A red check on the code needs a fix PR and a new
  build. Plan, apply and prod runs are never re-run by the agent (workflows.md § Commands).

## 5. What the build does not do

- It moves no `ENG` status: the integration already moved merged 2.0 issues to `QA`, and QA moves them
  on after checking.
- It moves no `FEAT` status. A feature whose fix this build carries goes `QA failed` → `Ready for QA`
  at "the sync when a fix reaches the stand" (`events-tracker-base` § Statuses → `FEAT`), which is not
  this skill. List such features in the QA post draft so Artem sees them; do not move them.
- It does not touch prod and does not make the build a release. A release is a later step, on a build
  QA has checked (`releasing`).
