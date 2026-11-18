# Prod before the first user

Loaded from `releasing` → `SKILL.md` step 4 (before the plan) and step 6 (after the apply, before
anyone is told). The workflows check their own preconditions (`../../../references/workflows.md`
§ Preconditions) and nothing else: none of what follows. 2.2.5 reached prod green on every family with
every service able to scale to zero; it was found the next morning, before the first user session.

The agent makes no cloud call. It reads what the repositories say prod should be, and hands Artem the
commands that read what prod is. Each line gets one of: holds, does not hold, not read.

## Before the plan: the ground

What the version needs from prod's ground (Contorra-Events-Cloud, `envs/prod/topology.yaml`), compared
with what prod already had at its last `prod/v` tag:

- **New secrets.** Every secret a family of this version mounts has a version on prod. A secret that a
  person writes (a Sentry DSN, the replay exclusion list) does not get one from the bootstrap.
- **New grants.** The bootstrap's `iam` dry run on prod lists nothing missing.
- **First deploys.** A family or resource that never ran on prod (as `edge-analytics` did not at 2.2.5)
  is deployed the first time by a person (D-038 in the workspace), or its prod values file says
  `DEPLOY=off` by a PR merged into the build.
- **Hosts.** Every public host of a family that is on has its domain mapping and is in Golf Pad's sign-in
  redirect list.

Read the topology and the merged PRs since the last `prod/v` tag; hand Artem the bootstrap dry runs
(`run.sh prod secrets`, `run.sh prod iam`) to run himself. Anything that does not hold goes to Artem
before the plan, not as a known issue: a missing piece of ground fails the apply halfway.

## Before the plan: what the forms say

From each family's prod values file at the build's commits (`<family>/deploy/env/prod.env` in Core,
`surfaces/<mcp|mcp/analytics|web|mobile>/deploy/env/prod.env` in Surfaces; the scaling and memory
settings arrive with Core #77 and Surfaces #239, so a build without them carries none):

- **Scaling.** `MIN_INSTANCES` is at least 1 for every service a user's first request reaches (core,
  identity, web-bff, edge-mcp; mobile-bff once the app uses it). A setting made by hand on prod is wiped
  by the next deploy, which renders the service from its form: if prod was fixed by hand, the values
  file must carry the same, or the apply undoes it.
- **Memory.** Core's `MEMORY` on prod (1 GiB from Core #77), and any family whose limit was hit on
  test.
- **The database.** The instance class and availability in the prod topology, and Core's connection
  pool against it. Today's floor is D-050 in the workspace (`db-g1-small`, no replica, until closer to
  parity): say what the topology says and whether it matches.

## After the apply, before anyone is told

Artem runs these (or says they were run); the agent reads the answers:

```
gcloud run services describe <service> --project <prod> --region <region> \
  --format='yaml(metadata.annotations,spec.template.metadata.annotations,spec.template.spec.containers[0].resources)'
gcloud sql instances describe <instance> --project <prod> --format='value(settings.tier,settings.availabilityType)'
```

- Every service runs the version (`/version` on the public hosts; Core by its revision label), with the
  scaling and memory the forms say. A minimum set on the service (`--min`, which Artem used on 11-18)
  shows in the service's own annotations, one set by the form in the template's: read both.
- The database is at the class D-050 names.
- A first real sign-in by a person (there is no prod test account).

Only then the team post, and only then the first user.
