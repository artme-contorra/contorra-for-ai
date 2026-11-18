# The CI workflows behind a test build and a release

As on `main` of `maivanov/Contorra-Events-Surfaces` at `7e8a1e1ac` (the merge of #233, after #226, #228,
#229 and #231) and of `maivanov/Contorra-Events-Core` at `55355e638` (the merge of #76, after #64, #73 and
#74): the commits of build `2.2.5`, the first one released through these workflows (2026-11-17).
This file is the one place in the plugin that names them: when a workflow changes a file name, an
input or an output, correct it here and bump the plugin. The two skills speak of workflows by role
(test-build, plan, apply) and never by file.

If a workflow named here is missing from the repository when a skill runs, the skill stops and says
so. It never falls back to deploying from a laptop.

## Repositories

| Short name | GitHub |
|---|---|
| Core | `maivanov/Contorra-Events-Core` |
| Surfaces | `maivanov/Contorra-Events-Surfaces` |
| Proto | `maivanov/Contorra-Proto` |

## The workflows by role

| Role | Lives in | File | Inputs | Who starts it | What it does |
|---|---|---|---|---|---|
| test-build | Surfaces | `.github/workflows/test-build.yml` | `stage` (optional: a new stage only on Artem's word; empty keeps the current one); `skipped` (optional: versions an earlier failed build spent and never tagged, space-separated, e.g. `2.2.4 2.2.5`; checked in compose, before anything deploys) | anyone on the team, or the agent through `cutting-a-test-build` | Composes `main` of Core and Surfaces at dispatch (same Proto pin, green `floor`, or it stops and says which), numbers the build `2.<stage>.<build>`, deploys to test through each repository's `deploy.yml` (Core first, then the Surfaces families; a repository whose `main` did not move since the previous build is carried, not rebuilt), runs the stand scenarios, then tags both repositories `v2.<stage>.<build>` with the build manifest. A build whose scenarios fail is still tagged (the manifest says so in `scenarios:`). A failed build retried with the same inputs while neither `main` moved takes the same number and deploys the images it already pushed. Dispatched from any ref but `main`, it does nothing and reports no error (`Compose` and everything after it is skipped) |
| plan | Surfaces | `.github/workflows/release-plan.yml` | `version` (`2.<stage>.<build>`); `known_issues` (optional, one line: the known issues Artem accepted; empty when there are none) | **Artem only** (the release owner); any other actor is refused | Prod dry run of every family for that version (Core's in two Core runs, each Surfaces family in its own job); reads the latest green nightly on a descendant of the build in each repository and shows it as "read, not gating" (it runs no nightly and waits for none); reads the build's scenario result; writes the gate and the known issues into the run's summary; its `Summary` job decides whether the plan passed. `Summary` fails on a failed dry run, or on red scenarios with an empty `known_issues`. It checks only that `known_issues` is not empty, not that it names the red part |
| apply | Surfaces | `.github/workflows/release-apply.yml` | `version` | **Artem only** (the release owner); any other actor is refused | Follows the latest completed run titled `Release plan · <version>`, and only if its `Summary` job passed: a newer failed plan overrides an older green one. Refuses otherwise. Deploys the version's images to prod in the order (the console is the exception: it is rebuilt from source at the tag, see below), checks the public hosts anonymously, then tags the release `prod/v<version>` on Core and Surfaces |

What the workflows call, already named by earlier steps:

- `deploy.yml` in Core and in Surfaces: dispatched or called by test-build (test target), plan (prod,
  dry run) and apply (prod). The skills never dispatch it directly.
- `floor`: a job of each repository's `ci.yml`. CI runs on pull requests (and by hand), not on a push
  to `main`, so a merge commit on `main` carries no `floor` of its own: the workflow reads the latest CI
  run on `main`'s commit or on its second parent (the head of the PR that made it). A squash merge or a
  direct push to `main` has no `floor` and is refused.
- `nightly.yml` on `main` of Core (cron `0 20 * * *` UTC, up to 6 hours) and Surfaces (`15 20 * * *`
  UTC), mutation included. A standing measure of the tests, not part of the release gate: the plan reads
  it and gates on none.
- `tooling/release/tag.sh` in Surfaces: numbers and tags Core and Surfaces as one pair (Proto gets no
  product version).
- `tooling/delivery/lib.sh` in Core and in Surfaces: the per-family build and deploy (§ Images).
- `surfaces/web/client/src/shared/flags/release.json` in Surfaces: the opened `FEAT` features.

The deploy order: Core (core, then identity, migrations with core), then the Surfaces families
edge-mcp, edge-analytics, web (the BFF, then the console), mobile. Every family deploys only through
these workflows, the mobile BFF included: it rides every build from `main`, and nobody deploys it by
hand beside them.

A family's values file for an environment may say `DEPLOY=off` (Surfaces #233): its `deploy.sh` then
says the family is off there and exits green before any cloud call. `edge-analytics` is off on test and
prod until its hosts and sign-in exist. Turning a family on is one line in its values file, plus the
ground it needs (§ Preconditions).

What the prod dry runs do not cover:

- **Core's migration job** has no server-side dry run (`job.yaml has no server-side dry run` in Core's
  dry-run log). Every migration between the version prod runs and the released one runs for the first
  time on prod data during the apply.
- **The console** has no dry run in the plan. At apply it is not promoted from test: the `web-console`
  job checks out the tag, runs `pnpm install` and builds it on the runner (`hosting.sh ... --apply`),
  checks it on a channel, then clones it to live. Prod gets a fresh build of the same source.

What the apply does not know:

- **`Public hosts` ignores `DEPLOY=off`.** It checks every public host, `analytics.golfpad.events`
  too, so an apply whose families are all green ends red at `Public hosts` and writes no `prod/v` tag
  while `edge-analytics` is off (2.2.5). Until the check skips an off family, read such a red as
  `releasing` step 5 says: prod runs the version, the tag is missing.
- **Scaling, memory and the database.** The forms are the same for test and prod unless a values file
  says otherwise, and a deploy replaces the service from its form, so a setting made by hand on prod
  (a minimum instance count) is wiped by the next deploy. 2.2.5 reached prod with every service able to
  scale to zero. What prod needs before people use it is in `releasing` → `prod-checklist.md`.

## Images: one manifest per image

Since Core #73 and Surfaces #228 every image is built with `--provenance=false --sbom=false`, so it is
pushed as one image manifest, and `single_manifest` refuses an index right after the push and on the
retry path (a version whose images are already pushed). A Cloud Run revision records the platform
manifest, so a deploy of an index fails its post-replace digest check after the service was already
replaced (test build 2.2.2's `Core → test`).

The check runs at build and retry only, not at a deploy of a version pushed earlier: a version pushed
before #73 / #228 is not checked when it is deployed. So:

- `2.2.0` and `2.2.1` cannot be released (also § Builds CI cannot carry);
- `2.2.2`, `2.2.3` and `2.2.4` are spent: each pushed images and was never tagged (the digest check,
  two network drops on the self-hosted runner, the test ground behind the build);
- a build that carries a family `behind` at a version pushed before #73 / #228 cannot be released.

## Preconditions the plan and apply check first

Nothing reaches prod until all of these hold. The plan and apply check the Surfaces ones in their
first job (`Guard`); Core's own variables stop only Core's dispatched run, so in the plan they show as a
failed `Dry run · core + identity` (second job), in the apply as a failed `core + identity`.

- the repository variable `RELEASE_OWNER` names Artem's account, in Surfaces and in Core (empty means
  nobody may run prod);
- the repository variables `PROD_WORKLOAD_IDENTITY_PROVIDER`, `PROD_DEPLOYER_EDGE`, `PROD_DEPLOYER_WEB`
  and `PROD_DEPLOYER_MOBILE` are set in Surfaces, and `PROD_WORKLOAD_IDENTITY_PROVIDER` and
  `PROD_DEPLOYER_CORE` in Core (prod's federation, an IAM change that is Artem's); the federation
  provider must admit the run (it admits only runs Artem started);
- the Surfaces secret `CROSS_REPO_TOKEN` acts as the release owner's account. The plan and apply
  dispatch Core's `deploy.yml` with it, and Core's prod guard checks `github.actor` and
  `triggering_actor` against Core's `RELEASE_OWNER` on that run: a token of any other account makes
  Core unreleasable, with Core's run refused at its `Guard`;
- the Surfaces secrets `GOLFPAD_TEST_EMAIL` and `GOLFPAD_TEST_PASSWORD` exist (the stand scenarios on
  the runner sign in with them; without them every build records `scenarios: failure`). Set on
  2026-11-17.

A stop with one of these messages is not a defect of the build: say which precondition is missing.

Not checked by any workflow: the **ground** of the target environment (Contorra-Events-Cloud's
topology: secrets with a version, runtime grants, a resource's first deploy, host mappings). A build
that carries a family whose ground is behind fails in that family's deploy: 2.2.4 on test found 8
provided secrets without a version, 14 missing grants and an `edge-analytics` that did not exist, all
from analytics and Sentry work merged since 2.2.1. The bootstrap in Contorra-Events-Cloud (`run.sh
<env> secrets|iam|...`, a dry run first) is a person's step: on test before the build, on prod before
the plan (`prod-checklist.md`).

## Runs and jobs, as `gh run list` and `gh run view --json jobs` name them

- **test-build**, run `Test build · current stage` (or `· stage <n>`): `Compose` (the Proto pin,
  `floor`, the number and the `skipped` check; it also stops on "no v2 tag yet, name the stage", "a
  tag on Surfaces but not on Core", and "neither Core nor Surfaces main moved"); `Core → test` (Core's
  deploy, a separate Core run whose URL it prints); `Surfaces → test / <family>` and
  `Surfaces → test / Scenarios` (when Surfaces moved; families `edge-mcp`, `edge-analytics`, `web-bff`,
  `mobile-bff`, `web-console`); `Scenarios` (when only Core moved); `Tag & manifest`.
- **plan**, run `Release plan · <version>`: `Guard`, `Dry run · core + identity`, `Dry run · <family> /
  ...` per Surfaces family, `Nightly · read, not gating`, `Summary`.
- **apply**, run `Release apply · <version>`: `Guard`, `core + identity`, `edge-mcp / ...`,
  `edge-analytics / ...`, `web / ...`, `mobile / ...`, `Public hosts`, `Tag`.
- **Core's deploys** (dispatched by the three above), run `Deploy <family> → <target> · <version>`
  (`· dry run` appended for a dry run): jobs `Guard`, `core`, `identity`. Surfaces' own `deploy.yml`
  runs are named the same way.

## Where the jobs run

Every job that talks to Google runs on GitHub's hosted runners (`ubuntu-24.04`): every job of both
`deploy.yml`, so the test deploys, the prod dry runs and the prod deploys; `Scenarios` of test-build;
`Public hosts` of apply (Core #74, Surfaces #231). The team's own machine dropped TLS connections to
Google's APIs twice in twenty minutes during build 2.2.3; this stays while it cannot hold them.

The jobs that talk only to GitHub stay on the one self-hosted runner of each repository (`gpeci-surfaces`,
`gpeci-core`): compose, the waits for Core's runs, `Tag & manifest`, `Tag`, the guards, the gate and
`Summary`, and the CI `floor` and the nightly. Jobs queue there, and the waits count queue time:
`Core → test` waits up to 6600 s, each Core dry run in the plan up to 3000 s, Core's apply up to
6600 s; on a timeout the waited Core run is cancelled. Core's nightly (from 20:00 UTC, up to 6 hours) or
a queue of Core PR CI runs can starve a plan or apply into that cancel.

The self-hosted runner keeps its checkout between runs, so whatever a CI `pnpm install` leaves there
stays: lefthook's `pre-push` hook made `Tag & manifest` of 2.2.5 abort its push with "Can't find lefthook
in PATH". That is the machine, not the build: say so, and the hook is set aside on the runner by a
person before the job is rerun.

## Outputs the skills read, and how

`gh run view` prints a run's jobs and annotations, not its step summaries. Read each output where it
can be read:

- **The build manifest**: in the annotated tag `v<version>` on Surfaces (Core carries the same):
  `gh api repos/<repo>/git/ref/tags/v<version> --jq .object.sha`, then
  `gh api repos/<repo>/git/tags/<that sha> --jq .message`. It holds the commits of Core and Surfaces,
  the Proto pin, the images (`deployed:`), the families carried from an older build (`behind:`), any
  `skipped:` versions, `scenarios:`, the run URLs and the PRs merged since the previous build
  (`merged:`). `merged:` reads each merge commit's branch and title only, not the PR description; Core's
  branches (`worktree/...`) and titles rarely name an `ENG` key, so `merged:` is not the list of
  `ENG` issues. `skipped:` writes each spent version as "pushed, never deployed"; read it as "spent,
  never tagged": a spent version may have been partly deployed (2.2.2's Core was replaced on test).
- **The release tag**: the annotated tag `prod/v<version>` on both repositories (read the same way):
  version, time, who, the plan and apply runs, Core's runs. The known issues are not in it: they are in
  the summary of the plan run it names, which is the release record. A `prod/v` tag is never moved: a second
  apply of a version already tagged deploys again and then fails at its tag step.
- **Outcomes**: `gh run view <id> --json conclusion,jobs` (each job's name and conclusion), and the
  `::error::` annotations (`gh run view <id>` lists them). The workflows' own stops are annotations; a
  failure inside a deploy script is not: it prints `[<label>] FAIL: ...` to the log, and the job ends
  with GitHub's generic "Process completed with exit code 1". For those read `--log-failed`, of the Core
  run as well when Core failed.
- **What a red deploy job already changed**: a deploy replaces a resource first and checks it after,
  so a red family job may have moved its service. In the job's log (Core: the Core run's log):
  `job <name> completed on <image>` means the migration job ran; `<kind> <name>: <revision> is ready,
  holds 100%, runs <image>` means the service moved and was confirmed; `deployed <family> <version>`
  means the family finished. A `FAIL` after the replace (`holds <image>, expected ...`, `is not the
  ready one`, `does not hold 100%`, `revision ... runs ..., expected ...`, `<job> did not complete`)
  means the resource was replaced and the check failed: count the family as moved. `was not replaced`,
  `jobs replace refused`, or a `FAIL` before any replace means it did not move.
- **A failed revision is not retried by a rerun.** When a new revision fails to start (a startup
  probe), the service keeps that failed revision as its latest. A deploy of the same version renders the
  same template, so Cloud Run starts nothing and the job reports the same failure again within a second
  (`Creating Revision....failed`, naming the old revision). A real second start needs a template that
  differs (a revision suffix), which is a person's step on test and Artem's on prod (2.2.5's
  `mobile-bff`, forced as `--revision-suffix=retry1`, then the job's rerun passed). Report it so; do not
  dispatch a retry that cannot start anything.
- **Per-family prod diffs and deploy output**: each Surfaces family job's log
  (`gh run view <id> --log --job <job id>`; `deploy.sh` output is tee'd there). Core's dry runs and
  deploys are separate Core runs: their URLs are in the plan's and apply's logs and in the release
  tag; read them there (`--log-failed` on the Surfaces run does not show them).
- **Summary tables** (the plan's gate, its nightly lines and known issues, the apply's host list, a
  test build's "Not tagged" note): only on the run page. Whether a plan passed is readable: its `Summary`
  job's conclusion. Ask Artem to open it, or say they were not read. Never infer their content.

## Commands

The agent may run these:

```
# start a test build (skipped: only the versions an earlier failed build spent)
gh workflow run test-build.yml --repo maivanov/Contorra-Events-Surfaces --ref main \
  [-f stage=<n>] [-f skipped="<2.x.y ...>"]

# find the run it started and wait for it
gh run list --workflow test-build.yml --repo maivanov/Contorra-Events-Surfaces --limit 3 \
  --json databaseId,status,conclusion,createdAt,url
gh run watch <run-id> --repo maivanov/Contorra-Events-Surfaces --exit-status

# read any run's jobs, annotations and logs, and the tags
gh run view <run-id> --repo maivanov/Contorra-Events-Surfaces [--json conclusion,jobs] [--log-failed]
gh run view <run-id> --repo maivanov/Contorra-Events-Surfaces --log --job <job-id>
gh api repos/maivanov/Contorra-Events-Surfaces/git/matching-refs/tags/v2. --jq '.[].ref'
gh api repos/maivanov/Contorra-Events-Surfaces/git/matching-refs/tags/prod/v2. --jq '.[].ref'

# what is queued on a repository's one runner, and its latest nightly
gh run list --repo <repo> --status in_progress --json name,displayTitle,url
gh run list --repo <repo> --status queued --json name,displayTitle,url
gh run list --workflow nightly.yml --repo <repo> --branch main --limit 3 \
  --json headSha,conclusion,url,createdAt
```

**The agent never runs these.** They are handed to Artem, who runs them himself:

```
gh workflow run release-plan.yml  --repo maivanov/Contorra-Events-Surfaces --ref main -f version=<2.x.y> \
  [-f known_issues="<the line Artem accepted>"]
gh workflow run release-apply.yml --repo maivanov/Contorra-Events-Surfaces --ref main -f version=<2.x.y>
```

Nor does the agent ever run `gh run rerun`, `gh run cancel`, or `gh workflow run deploy.yml` on a
release-plan, release-apply or prod deploy run (Core's dispatched prod runs included). A rerun or a
cancel of those is handed to Artem like the start. A cancelled plan or apply does not cancel a Core
run it dispatched: after a cancel, read Core's `Deploy ... → prod · <version>` runs and say whether one
is still going.

Their pages: `https://github.com/maivanov/Contorra-Events-Surfaces/actions/workflows/release-plan.yml`
and `.../release-apply.yml`. GitHub's "Run workflow" form does not take input values from the URL, so
the link opens the form and Artem types the version (and the known issues); the command above carries
them filled in.

The workflows refuse any `github.actor` or `triggering_actor` but the release owner (the repository
variable `RELEASE_OWNER`, Artem's account), in every job that can reach prod, so a rerun of one job by
another account fails at that job's first step. That check cannot
tell Artem from an agent using Artem's `gh` login, so the rule that an agent never starts, reruns or
cancels plan or apply holds in the skills, not in the workflow.

## Builds CI cannot carry

`v2.2.0` and `v2.2.1` were cut before these workflows. At those tags Surfaces' `deploy.sh` takes no
version argument, so a CI deploy at those commits refuses it, and their images predate the
single-manifest check (§ Images): neither version can be released through the plan and apply, and a
build that carries a Surfaces family `behind` at `2.2.1` or older cannot either. `2.2.2` to `2.2.4` are
spent. `2.2.5` is the first build released through them (2026-11-17).

## Open (not decided yet)

Artem's open questions. A step that depends on one asks him; it does not choose.

- `--long` stand scenarios: neither test-build nor `deploy.yml` passes `--long`. Whether a release
  needs them.
- The release record: the known issues live only in the plan run's summary, which goes with the run's
  retention; the `prod/v` tag points at the run but does not carry them. Whether they go into the tag.
- Rollback: the previous version's images stay; a one-command rollback is not part of these workflows
  yet, and a rollback to a version pushed before #73 / #228 is not checked (§ Images).
