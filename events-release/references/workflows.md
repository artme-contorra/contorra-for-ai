# The CI workflows behind a test build and a release

As in `maivanov/Contorra-Events-Surfaces` PR #226 (branch `ak/release-workflows`, read at `31a7971ee`).
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
| test-build | Surfaces | `.github/workflows/test-build.yml` | `stage` (optional: a new stage only on Artem's word; empty keeps the current one); `skipped` (optional: versions an earlier failed build spent and never tagged, space-separated, e.g. `2.2.4 2.2.5`; checked in compose, before anything deploys) | anyone on the team, or the agent through `cutting-a-test-build` | Composes `main` of Core and Surfaces at dispatch (same Proto pin, green `floor`, or it stops and says which), numbers the build `2.<stage>.<build>`, deploys to test through each repository's `deploy.yml` (Core first, then the Surfaces families; a repository whose `main` did not move since the previous build is carried, not rebuilt), runs the stand scenarios, then tags both repositories `v2.<stage>.<build>` with the build manifest. A build whose scenarios fail is still tagged (the manifest says so in `scenarios:`). A failed build retried with the same inputs while neither `main` moved takes the same number and deploys the images it already pushed |
| plan | Surfaces | `.github/workflows/release-plan.yml` | `version` (`2.<stage>.<build>`); `known_issues` (optional, one line: the known issues Artem accepted; empty when there are none) | **Artem only** (the release owner); any other actor is refused | Prod dry run of every family for that version (Core's in two Core runs, each Surfaces family in its own job); reads the nightly gate and the build's scenario result; writes the gate and the known issues into the run's summary; its `Summary` job decides whether the plan passed. With no green nightly on the build's commit or a descendant, it runs the nightly on `main` (a descendant of the build), which can take hours. `Summary` fails on a failed dry run, or on a red part (scenarios, a nightly) with no `known_issues`. The run as a whole can be red (a nightly it ran failed) while `Summary` passed on known issues; the apply goes by `Summary` |
| apply | Surfaces | `.github/workflows/release-apply.yml` | `version` | **Artem only** (the release owner); any other actor is refused | Follows the latest completed run titled `Release plan · <version>`, and only if its `Summary` job passed: a newer failed plan overrides an older green one. Refuses otherwise. Deploys the same images to prod in the order, checks the public hosts anonymously, then tags the release `prod/v<version>` on Core and Surfaces |

What the workflows call, already named by earlier steps:

- `deploy.yml` in Core and in Surfaces: dispatched or called by test-build (test target), plan (prod,
  dry run) and apply (prod). The skills never dispatch it directly.
- `floor`: a job of each repository's `ci.yml`. CI runs on pull requests (and by hand), not on a push
  to `main`, so a merge commit on `main` carries no `floor` of its own: the workflow reads the latest CI
  run on `main`'s commit or on its second parent (the head of the PR that made it). A squash merge or a
  direct push to `main` has no `floor` and is refused.
- `nightly.yml` on `main` of Core and Surfaces, mutation included. A release needs it green.
- `tooling/release/tag.sh` in Surfaces: numbers and tags Core and Surfaces as one pair (Proto gets no
  product version).
- `surfaces/web/client/src/shared/flags/release.json` in Surfaces: the opened `FEAT` features.

The deploy order: Core (core, then identity, migrations with core), then the Surfaces families
edge-mcp, edge-analytics, web (the BFF, then the console), mobile.

## Preconditions the plan and apply check first

Both stop in their first job, and nothing reaches prod, until:

- the repository variable `RELEASE_OWNER` names Artem's account, in Surfaces and in Core (empty means
  nobody may run prod);
- the repository variables `PROD_WORKLOAD_IDENTITY_PROVIDER`, `PROD_DEPLOYER_EDGE`, `PROD_DEPLOYER_WEB`
  and `PROD_DEPLOYER_MOBILE` are set in Surfaces (prod's federation, an IAM change that is Artem's);
- Core's `deploy.yml` on `main` has a `prod` target and a `dry_run` input (on 2026-11-17 it offers
  `test` only; Core's release PR adds them).

A stop with one of these messages is not a defect of the build: say which precondition is missing.

## Runs and jobs, as `gh run list` and `gh run view --json jobs` name them

- **test-build**, run `Test build · current stage` (or `· stage <n>`): `Compose` (the Proto pin,
  `floor`, the number and the `skipped` check; it also stops on "no v2 tag yet, name the stage", "a
  tag on Surfaces but not on Core", and "neither Core nor Surfaces main moved"); `Core → test` (Core's
  deploy, a separate Core run whose URL it prints); `Surfaces → test / <family>` and
  `Surfaces → test / Scenarios` (when Surfaces moved; families `edge-mcp`, `edge-analytics`, `web-bff`,
  `mobile-bff`, `web-console`); `Scenarios` (when only Core moved); `Tag & manifest`.
- **plan**, run `Release plan · <version>`: `Guard`, `Dry run · core + identity`, `Dry run · <family> /
  ...` per Surfaces family, `Nightly gate`, `Nightly · start Core`, `Nightly · Surfaces`,
  `Nightly · wait for Core`, `Summary`.
- **apply**, run `Release apply · <version>`: `Guard`, `core + identity`, `edge-mcp / ...`,
  `edge-analytics / ...`, `web / ...`, `mobile / ...`, `Public hosts`, `Tag`.
- **Core's deploys** (dispatched by the three above), run `Deploy <family> → <target> · <version>`
  (`· dry run` appended for a dry run): jobs `Guard`, `core`, `identity`. Surfaces' own `deploy.yml`
  runs are named the same way.

## Outputs the skills read, and how

`gh run view` prints a run's jobs and annotations, not its step summaries. Read each output where it
can be read:

- **The build manifest**: in the annotated tag `v<version>` on Surfaces (Core carries the same):
  `gh api repos/<repo>/git/ref/tags/v<version> --jq .object.sha`, then
  `gh api repos/<repo>/git/tags/<that sha> --jq .message`. It holds the commits of Core and Surfaces,
  the Proto pin, the images (`deployed:`), the families carried from an older build (`behind:`), any
  `skipped:` versions, `scenarios:`, the run URLs and the PRs merged since the previous build with
  their `ENG` keys (`merged:`).
- **The release tag**: the annotated tag `prod/v<version>` on both repositories (read the same way):
  version, time, who, the plan and apply runs, Core's runs. The known issues are not in it: they are in
  the summary of the plan run it names, which is the release record. A `prod/v` tag is never moved: a second
  apply of a version already tagged deploys again and then fails at its tag step.
- **Outcomes**: `gh run view <id> --json conclusion,jobs` (each job's name and conclusion), and the
  `::error::` annotations (every stop in these workflows is one; `gh run view <id>` lists them).
- **Per-family prod diffs and deploy output**: each Surfaces family job's log
  (`gh run view <id> --log --job <job id>`; `deploy.sh` output is tee'd there). Core's dry runs and
  deploys are separate Core runs: their URLs are in the plan's and apply's logs and in the release
  tag; read them there (`--log-failed` on the Surfaces run does not show them).
- **Summary tables** (the plan's gate and known issues, the apply's host list, a test build's "Not
  tagged" note): only on the run page. Whether a plan passed is readable: its `Summary` job's
  conclusion. Ask Artem to open it, or say they were not read. Never infer
  their content.

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
version argument, so a CI deploy at those commits refuses it: neither version can be released through
the plan and apply, and a build that carries a Surfaces family `behind` at `2.2.1` or older cannot
either. The next build from the current `main` is clean.

## Open (not decided yet)

- Mobile BFF: plan and apply deploy the `mobile` family as the last one. Whether Denis keeps a hand
  deploy beside it is open.
- Rollback: the previous version's images stay; a one-command rollback is not part of these workflows
  yet.
