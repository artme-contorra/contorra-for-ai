# The CI workflows behind a test build and a release

**Provisional.** The test-build, plan and apply workflows do not exist yet. They are being built in
`maivanov/Contorra-Events-Surfaces`, and the file names and inputs below are the expected ones, not
read from a repository. This file is the one place in the plugin that names them: when a workflow
lands with another file name, input or output, correct it here and bump the plugin. The two skills
speak of workflows by role (test-build, plan, apply) and never by file.

If a workflow named here is missing from the repository when a skill runs, the skill stops and says
so. It never falls back to deploying from a laptop.

## Repositories

| Short name | GitHub |
|---|---|
| Core | `maivanov/Contorra-Events-Core` |
| Surfaces | `maivanov/Contorra-Events-Surfaces` |
| Proto | `maivanov/Contorra-Proto` |

## The workflows by role

| Role | Lives in | File (provisional) | Inputs (provisional) | Who starts it | What it does |
|---|---|---|---|---|---|
| test-build | Surfaces | `.github/workflows/test-build.yml` | `stage` (optional: a new stage only on Artem's word; default the current one) | anyone on the team, or the agent through `cutting-a-test-build` | Composes `main` of Core and Surfaces at dispatch (same Proto pin, green `floor`, or it stops and says which), numbers the build `2.<stage>.<build>` with `tooling/release/tag.sh` and tags both repositories, builds each family once, deploys to test through each repository's `deploy.yml` in order, runs the stand scenarios, records the build manifest |
| plan | Surfaces | `.github/workflows/release-plan.yml` | `version` (`2.<stage>.<build>`) | **Artem only**; any other actor is refused | Prod dry-run of every family for that version; writes the diff, the gate and the known issues into the run's summary. With no green nightly on the build's commits or a descendant, runs the nightly suite on the build's commits first |
| apply | Surfaces | `.github/workflows/release-apply.yml` | `version` | **Artem only**; any other actor is refused | Deploys the same images to prod in the order, checks the public hosts anonymously, tags the release |

What the workflows call, already named by earlier steps:

- `deploy.yml` in Core and in Surfaces: dispatched by test-build (test target) and by apply (prod target,
  to come). The skills never dispatch it directly.
- `floor`: the required status of a PR in Core and Surfaces. A red `floor` on `main` stops a test build.
- The nightly run on `main` of Core and Surfaces, mutation included. A release needs it green.
- `tooling/release/tag.sh` in Surfaces: numbers and tags Core and Surfaces as one pair (Proto gets no
  product version).
- `surfaces/web/client/src/shared/flags/release.json` in Surfaces: the opened `FEAT` features.

The deploy order is the repositories' own: core and identity, migrations with core, then the surfaces
and the BFFs.

## Outputs the skills read (provisional)

- **The build manifest**: commits of Core and Surfaces, the Proto pin, images, the scenario result, and
  the PRs merged since the previous build with their `ENG` issues. Expected in the annotated tag
  `v2.<stage>.<build>` and in the test-build run's summary.
- **The plan's summary**: per family the prod diff, then the gate and the known issues for the version.
- **The apply's summary**: what was deployed, the host checks, the tag pushed.

## Commands

The agent may run these:

```
# start a test build
gh workflow run test-build.yml --repo maivanov/Contorra-Events-Surfaces --ref main [-f stage=<n>]

# find the run it started and wait for it
gh run list --workflow test-build.yml --repo maivanov/Contorra-Events-Surfaces --limit 3 \
  --json databaseId,status,conclusion,createdAt,url
gh run watch <run-id> --repo maivanov/Contorra-Events-Surfaces --exit-status

# read any run's summary and logs, and the tags
gh run view <run-id> --repo maivanov/Contorra-Events-Surfaces [--log-failed]
gh api repos/maivanov/Contorra-Events-Surfaces/git/matching-refs/tags/v2. --jq '.[].ref'
```

**The agent never runs these.** They are handed to Artem, who runs them himself:

```
gh workflow run release-plan.yml  --repo maivanov/Contorra-Events-Surfaces --ref main -f version=<2.x.y>
gh workflow run release-apply.yml --repo maivanov/Contorra-Events-Surfaces --ref main -f version=<2.x.y>
```

Their pages: `https://github.com/maivanov/Contorra-Events-Surfaces/actions/workflows/release-plan.yml`
and `.../release-apply.yml`. GitHub's "Run workflow" form does not take input values from the URL, so
the link opens the form and Artem types the version; the command above carries it filled in.

The workflows refuse any `github.actor` but Artem's account. That check cannot tell Artem from an agent
using Artem's `gh` login, so the rule that an agent never starts plan or apply holds in the skills, not
in the workflow.

## Open (not decided yet)

- Mobile BFF: Denis deploys it himself today; whether it joins the ordered deploy is open.
- Rollback: the previous version's images stay; a one-command rollback is not part of these workflows
  yet.
- Prod from CI needs the deployers' federation in Events Prod (an IAM change, Artem's to run). Until it
  is in place, plan and apply cannot reach prod.
