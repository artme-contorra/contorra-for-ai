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
  (`get_issue`). Read the description with `gh pr view <n> --repo <repo> --json title,body,headRefName`:
  the manifest's `merged:` reads only the branch and title, and Core's branches (`worktree/...`) and
  titles seldom carry a key (workflows.md § Outputs). On 2.0 a merged issue should be in `QA`; one that
  is not is a finding for the issue's owner (the PR did not name it, or the integration is not
  connected for that repository).
- Proto: list what merged there too, but Proto is carried by the pin, not built.

Say what the build is for. Typical answers: "the fixes for ENG-231 and ENG-235 that QA is waiting on";
"FEAT-12 opens (release.json changed in #230)"; "a week of small fixes, nothing QA is waiting on, can
wait". The last one is a reason not to build.

## 2. What would stop the workflow

- **The Proto pin.** Core and Surfaces must pin the same Proto commit at their `main`. Read each
  `proto` submodule entry: `gh api repos/<repo>/contents/proto?ref=main --jq .sha`. Different pins: the
  side that is behind needs its pin PR merged first; say which. Any merge that moves the pin in one
  repository blocks every build until the other follows, and only compose shows it (Surfaces moved to
  `df5f43b` with the mobile BFF's PRs, Core followed with #76 before 2.2.5): when § 1 finds a pin move
  on one side, say the build waits on the other side's pin PR.
- **The test ground.** No workflow checks that test's ground (Contorra-Events-Cloud's topology) holds
  what the build's families need. If § 1 found PRs that add a secret, a grant, a family or a host
  (analytics, Sentry and replay did, and 2.2.4 failed on them), ask the person to run the test
  bootstrap's dry run (`run.sh test secrets|iam`) and close what it lists before the dispatch
  (`workflows.md` § Preconditions). A family that cannot deploy green yet is `DEPLOY=off` in its test
  values file, by a PR, not left to fail.
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

- **After a failed build: classify, then retry or skip.** Read the previous test-build run
  (`gh run list`, then `gh run view <id> --json conclusion,jobs`). If it ended untagged (its
  `Tag & manifest` job failed or never ran, and no `v<version>` tag exists for it), first classify the
  failure as § 4 says: machine or code. Then:
  - **A code failure** (a check that fails again on the same commits, as 2.2.2's digest check did) is
    never retried, even while neither `main` moved: say so, and the build waits for the fix PR on
    `main`. If the run pushed an image of its version, that version is spent (below).
  - **A machine failure, and neither Core's nor Surfaces' `main` moved:** retry once with the same
    inputs (the same `stage`, the same `skipped`, or none). The build takes the same number again and
    deploys the images already pushed as that version, without rebuilding them (both repositories,
    the single-manifest check included: workflows.md § Images).
  - **A `main` moved, and the failed run pushed an image of its version:** that version is spent. Pass
    it, with every version spent before it since the last tag, as `skipped`, space-separated
    (`-f skipped="2.2.2 2.2.3"`, as 2.2.4 was dispatched). A failed run whose images are pushed holds
    both repositories at its number: while neither `main` moves, only a retry with that number works,
    and a new number needs a new commit on each `main` that pushed an image of the spent one. Say which
    repository needs a commit.
  - When unsure which kind it is, say so and ask Artem; do not dispatch.
  - The run's "Not tagged" note says the same and names the value; it is on the run page only, so ask
    Artem to read it out, or derive the version: one past the highest `v2.<stage>.*` tag, or one past
    the highest `skipped` the run was given. Whether an image was pushed: the build job of that family
    succeeded (Core: `Core → test`'s Core run; Surfaces: `Surfaces → test / <family>`).
  - Compose checks `skipped` before anything deploys: a version that is not a gap from the next
    untagged number on stops the run with nothing deployed.
- Dispatch the test-build workflow from `main` of Surfaces (workflows.md § Commands), always with
  `--ref main`: from any other ref the run skips everything and reports no error. Pass `stage` only
  when Artem named a new one, `skipped` only as above. The run is titled `Test build · current stage`
  (or `· stage <n>`).
- Before the dispatch, look at what is queued on each repository's one runner (workflows.md § One
  runner per repository). A long queue on Core, or Core's nightly window (from 20:00 UTC), can cancel
  `Core → test` on its wait: say so and propose another time rather than dispatching into it.
- Find the run it started (`gh run list`, the newest `workflow_dispatch` run created after the
  dispatch) and give its URL to the person who asked.
- Watch it to the end (`gh run watch --exit-status`). No notice to QA before or during the run: QA is
  told about a build by Artem, from the post drafted after it (§ 4 and `qa-post.md`).
- Do not start a second test build while one is running.

## 4. Reading the result

- **Green.** Read the manifest from the tag `v<version>` (workflows.md § Outputs; the run's summary
  is not readable with `gh run view`): version, commits, Proto pin, images, `behind:`, `scenarios:`,
  merged PRs and their `ENG` issues. Check the version tag exists on both Core and Surfaces.
- **Red.** `gh run view <id> --json conclusion,jobs`, the run's `::error::` annotations, then
  `--log-failed`. Name the job (workflows.md § Runs and jobs): `Compose` (the reads of § 2 found
  late, the numbering, a `skipped` that is no gap, or "nothing moved"), `Core → test` (a separate Core
  run, `Deploy all → test · <version>`: follow its URL in the job's log and read that run's
  `--log-failed` too, since a deploy-script failure there is a `FAIL:` line, not an annotation),
  `Surfaces → test / <family>`, the scenarios, or `Tag & manifest`. Give the first failing line in
  plain words. A failure before the tagging leaves the version untagged: say so, and whether the next
  dispatch is a retry with the same inputs or carries it as `skipped` (§ 3).
- **Which families moved.** A red family job does not mean its family stayed: a deploy replaces the
  service (and runs Core's migration job) before it checks it, so the service may already run the
  new build. Read each deploy job's log, red ones included, by the lines in workflows.md § Outputs
  ("What a red deploy job already changed"), and say per family: moved and confirmed, moved but
  unconfirmed, or not moved. Test is then mixed (2.2.2 left Core and its migrations on the new build,
  identity and Surfaces on the previous one). There is no rollback. Draft a stand-status line for
  Artem (`qa-post.md` § Stand status), so QA knows what the stand runs; he decides whether to send it.
- **Machine or code.** Classify every red job before deciding the next dispatch:
  - machine: the runner restarted or lost its network (a TLS drop to `sts.googleapis.com` or
    `iam.googleapis.com` in `google-github-actions/auth`, as 2.2.3 twice), the registry or GitHub did
    not answer, a wait timed out in the queue and cancelled the Core run, an authentication or clone
    failure on the runner, a git hook left on the self-hosted runner ("Can't find lefthook in PATH" in
    `Tag & manifest`, `workflows.md` § Where the jobs run). The same commits would pass on a healthy
    machine.
  - ground: a secret without a version, a missing grant, a resource that does not exist on test
    (2.2.4). Not the code and not the machine: a person runs the bootstrap (§ 2), then a retry.
  - a revision that did not start (a startup probe failure). A retry with the same version starts
    nothing (`workflows.md` § Outputs, "A failed revision is not retried by a rerun"): say so and ask
    the person to force a new revision on test; once it is ready, the run's failed jobs are rerun.
  - code: a check in the workflow or the deploy scripts that fails on these commits (a digest that
    does not match, a form that does not render, a failing scenario). It fails again on a retry.
  Say which, and why, in one line.
- **Tagged, scenarios red.** The run is red but the build is tagged and on test: the tagging job
  stops after the tag with "deployed and tagged, but the scenarios on test are ...". The build is not
  fit for QA and cannot be released: say so and do not draft a QA post for it.
- Never retry a red run blindly. On a test build only: a machine failure is retried once by
  dispatching again with the same inputs (§ 3), and said so. A code failure needs a fix PR and a new
  build. Plan, apply and prod runs are never re-run by the agent (workflows.md § Commands).

## 5. What the build does not do

- It moves no `ENG` status: the integration already moved merged 2.0 issues to `QA`, and QA moves them
  on after checking.
- It moves one `FEAT` status, after a green build and before the QA post (D-049 in the workspace): a
  feature that is `QA failed` and whose fix this build carries goes to `Ready for QA`. Say which and
  why (the PR and the `ENG` issue that fix it), move them, echo the result, and name them in the QA
  post. Nothing else: `QA passed` and `QA failed` are QA's, `Released` is `releasing`'s.
- It does not touch prod and does not make the build a release. A release is a later step, on a build
  QA has checked (`releasing`).
