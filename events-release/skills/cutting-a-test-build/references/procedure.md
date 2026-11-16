# Cutting a test build: the procedure

Loaded from `cutting-a-test-build` → `SKILL.md`. Workflow names, inputs and commands are in
`../../../references/workflows.md` (provisional).

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
- **`floor` on `main`.** The latest commit of `main` in each repository must have a green `floor`:
  `gh api repos/<repo>/commits/main/check-runs --jq '.check_runs[] | select(.name=="floor") | .conclusion'`.
  Red or missing: say which repository and stop.
- **The stage.** The default is the current stage. A new stage is Artem's call; ask, never assume.

## 3. Dispatch and wait

- Dispatch the test-build workflow from `main` of Surfaces (workflows.md § Commands). Pass `stage`
  only when Artem named a new one.
- Find the run it started (`gh run list`, the newest `workflow_dispatch` run created after the
  dispatch) and give its URL.
- Watch it to the end (`gh run watch --exit-status`). The run deploys to the shared test stand; tell
  the person that test is changing while it runs, since QA may be on it.
- Do not start a second test build while one is running.

## 4. Reading the result

- **Green.** Read the manifest (workflows.md § Outputs): version, commits, Proto pin, images,
  scenario verdict, merged PRs and their `ENG` issues. Check the version tag exists on both Core and
  Surfaces.
- **Red.** `gh run view <id> --log-failed`. Name the step (compose, number, build, deploy, scenarios)
  and the first failing line, in plain words. Compose failures are the reads of § 2 found late. A
  deploy failure leaves test on the previous build or partly updated: say which families moved. A
  scenario failure means the build is on test but not fit for QA: say so in the result and do not
  draft a QA post for it.
- A number the workflow pushed but never deployed is the workflow's to record; the skill only reports it.
- Never re-run a red run blindly. Re-run only a step that failed on the machine (network, a runner
  restart), once, and say so. A red check on the code needs a fix PR and a new build.

## 5. What the build does not do

- It moves no `ENG` status: the integration already moved merged 2.0 issues to `QA`, and QA moves them
  on after checking.
- It moves no `FEAT` status. A feature whose fix this build carries goes `QA failed` → `Ready for QA`
  at "the sync when a fix reaches the stand" (`events-tracker-base` § Statuses → `FEAT`), which is not
  this skill. List such features in the QA post draft so Artem sees them; do not move them.
- It does not touch prod and does not make the build a release. A release is a later step, on a build
  QA has checked (`releasing`).
