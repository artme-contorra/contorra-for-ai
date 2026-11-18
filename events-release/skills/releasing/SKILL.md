---
name: releasing
tools:
  - Read
  - Bash
  - mcp__linear-server__list_issues
  - mcp__linear-server__get_issue
  - mcp__linear-server__list_comments
  - mcp__linear-server__save_issue
description: >
  Take a checked test build of Events 2.0 to prod. Reads the build's scope
  (the FEAT features it carries) and the prod gate, shows Artem the gate and
  the known issues it would record, hands him the plan and apply workflows
  with the version filled in (only Artem starts them), reads their results,
  then moves the scope's FEAT features to `Released` and drafts the team
  post. Use whenever someone asks whether a build can go to prod, wants to
  release a version, asks what blocks a release, or says a release is done
  and the tracker and the team need to catch up. The agent never starts a
  prod workflow and sends nothing.
  Trigger phrases: "релиз", "выкатываем на прод", "можно на прод?", "что
  блокирует релиз", "релиз прошёл", "release 2.x.y", "ship to prod",
  "is the build ready for prod".
---

A release is a test build that passed, promoted unchanged: the same images, the same version (the console alone is rebuilt from the same source at the tag, `workflows.md` § The workflows by role). This skill carries the judgement (is the gate green, which known issues go on record, does the plan show what the scope predicts) and hands the two prod actions to Artem. The plan and apply workflows carry the mechanics; their names, inputs and commands are in `../../references/workflows.md` (the plugin's `references/`; the one place they are named).

## Steps

1. **Name the build.** The version `2.<stage>.<build>` of a test build the team has checked. If none is named, take the latest `v2.*` tag whose manifest says `scenarios: success` and confirm it with the person. A release always names a version that is already on test. `v2.2.0` and `v2.2.1` cannot be released through CI, `2.2.2` is spent, and a build carrying a family `behind` at `2.2.1` or older, or at any version pushed before the single-manifest fix, cannot be released either (`workflows.md` § Images, § Builds CI cannot carry): say so and propose the next build instead.
2. **Read the scope and the gate.** The scope is the `FEAT` features the build carries (its `release.json` and its manifest). The gate has four parts: the build's scenarios on test passed (`scenarios: success` in the tag's manifest); the prod dry runs pass (the plan's, read in step 4); every feature in scope `QA passed`; no open blocker in `ENG` `Triage`. The nightly (mutation included) is a standing measure of the tests, not part of the gate: report its latest state in one line, as the plan does ("read, not gating"). Show Artem each part with its state, and the known issues the release would record. How to read each: `references/gate.md`.
3. **A red gate stops the release**, unless Artem names the exception. An exception is a written known issue: what, why it is not a blocker, who accepted it. The agent proposes the wording; Artem accepts it. A failed prod dry run has no exception: it needs a fix (`gate.md` Part 3).
4. **Hand Artem the plan.** The link and the command with the version filled in, and the known issues Artem accepted in step 3 joined into one line as `known_issues` (each: `ENG-N` or `FEAT-N`, why not a blocker, accepted by; none: leave it out). The plan's summary is the release record, so a Linear-side exception (a feature not `QA passed`, a Triage blocker) goes in that line too; with red scenarios and no `known_issues` the plan's `Summary` job fails and the apply refuses the version (the workflow checks only that the line is not empty; that it names every red part is this skill's job). Before handing it over, put Artem's open questions for this release to him, say what is queued on the self-hosted runners, and go through the ground and the forms of `references/prod-checklist.md` (secrets, grants, first deploys, hosts; scaling, memory, the database): the workflows check none of it (`gate.md` § Before the plan). Artem starts it (`workflows.md` § Commands). When he says it ran (the run `Release plan · <version>`), read its outcome and logs (`workflows.md` § Outputs). The plan passed when its `Summary` job passed. Then: Core's prod diff in its two Core runs, each Surfaces family's in its job's log, against what the scope predicts; the console has no dry run in the plan (the apply checks it on a channel first), so say it was not previewed, and that prod gets a fresh build of it. Core's migration job has no dry run either: list the migrations that run for the first time on prod data during the apply (`gate.md` § Reading the plan). The gate table and known issues are in the run's summary, on the run page only: ask Artem to read them there, or say they were not read. Anything the scope does not explain (a family changing that should not, a migration nobody expected) is said plainly, and the release waits on Artem's call.
5. **Hand Artem the apply**, the same way, after he has read the plan. Artem starts it. When he says it ran (the run `Release apply · <version>`), read its outcome: every family job green, `Public hosts` green, `Tag` green and `prod/v<version>` on both repositories (`workflows.md` § Outputs). `Public hosts` red only on the host of a family that is `DEPLOY=off` (as `analytics.golfpad.events` at 2.2.5) is the check's known gap, not a failed release: every family job green means prod runs the version; say that the `prod/v` tag was not written and that the version's other hosts answered it. A family job that failed to start its revision is not fixed by a rerun (`workflows.md` § Outputs): the new revision is Artem's to force, then the failed jobs are rerun. Any other red apply is reported job by job: which families moved, what prod runs now (a red family job may have replaced its service: read its log as `workflows.md` § Outputs says), and that no `prod/v` tag was written. Prod then stays mixed, with no rollback in the workflows. Propose the next step: a fix PR, a new test build, a new plan and apply of that build; and, if users can see the mixed state, a draft notice for Artem. The same runner check as for the plan applies before Artem starts it. The apply follows the latest completed plan of the version; if it refuses, say whether that plan's `Summary` failed. After a cancel, check Core's `Deploy ... → prod · <version>` runs, which a cancel does not stop. Never re-run it or any job of it; a rerun is Artem's, like the start. A `prod/v` tag is never moved, so a second apply of an already tagged version deploys and then fails at its tag. The first real sign-in on prod is Artem's (there is no prod test account).
6. **After a green apply**: read prod against `references/prod-checklist.md` § After the apply, move the whole scope's `FEAT` features to `Released`, and draft the team post. `references/after-apply.md`.

## Rules

- **Only Artem starts plan and apply.** The agent hands links and commands and never runs them, even if its `gh` login could. It also never runs `gh run rerun` or `gh run cancel` on a plan, apply or prod deploy run (Core's dispatched runs included), nor `gh workflow run deploy.yml`: a rerun is handed to Artem like the start. The workflows refuse every account but the release owner (Artem's, set in the repositories' `RELEASE_OWNER` variable), in every prod job, but they cannot tell Artem from an agent using his login; this rule is what holds.
- **No cloud call by any route.** Everything that touches prod runs in the workflows. No deployer shell, no `gcloud`, no desk deploy as a fallback. If a workflow is missing or cannot reach prod (it stops until `RELEASE_OWNER`, prod's federation variables in both repositories and a `CROSS_REPO_TOKEN` acting as the release owner are in place, `workflows.md` § Preconditions), stop and report which.
- **The agent sends nothing.** The team post is a draft for Artem.
- **Tracker writes are limited to `FEAT` `Released`**, after a green apply, on Artem's yes to the list. No `ENG` status moves: on 2.0 they are Linear's GitHub integration's and QA's (`events-tracker-base` § Statuses → ENG flow on 2.0). No mark is set by hand. Linear Releases are not used.

## References

- `references/gate.md`: reading the scope and each part of the gate, blockers, the known-issue record.
- `references/prod-checklist.md`: prod before the first user, what no workflow checks: the ground, scaling, memory, the database.
- `references/after-apply.md`: the `Released` moves and the team post draft.
- `../../references/workflows.md`: the workflows by role, their names, inputs, jobs and outputs, the images, the preconditions, the runners, the builds CI cannot carry, the commands, the open questions.
