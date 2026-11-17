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

A release is a test build that passed, promoted unchanged: the same images, the same version. This skill carries the judgement (is the gate green, which known issues go on record, does the plan show what the scope predicts) and hands the two prod actions to Artem. The plan and apply workflows carry the mechanics; their names, inputs and commands are in `../../references/workflows.md` (the plugin's `references/`; the one place they are named).

## Steps

1. **Name the build.** The version `2.<stage>.<build>` of a test build the team has checked. If none is named, take the latest `v2.*` tag whose manifest says `scenarios: success` and confirm it with the person. A release always names a version that is already on test. `v2.2.0` and `v2.2.1` cannot be released through CI, nor a build carrying a Surfaces family `behind` at `2.2.1` or older (`workflows.md` § Builds CI cannot carry): say so and propose the next build instead.
2. **Read the scope and the gate.** The scope is the `FEAT` features the build carries (its `release.json` and its manifest). The gate has four parts: the build's scenarios on test passed (`scenarios: success` in the tag's manifest); every feature in scope `QA passed`; no open blocker in `ENG` `Triage`; a green nightly on the build's commit or a descendant in Core and in Surfaces. Show Artem each part with its state, and the known issues the release would record. How to read each: `references/gate.md`.
3. **A red gate stops the release**, unless Artem names the exception. An exception is a written known issue: what, why it is not a blocker, who accepted it. The agent proposes the wording; Artem accepts it. A missing nightly is not red: the plan runs the nightly on `main` (a descendant of the build) first, which can take hours.
4. **Hand Artem the plan.** The link and the command with the version filled in, and the known issues Artem accepted in step 3 joined into one line as `known_issues` (each: `ENG-N` or `FEAT-N`, why not a blocker, accepted by; none: leave it out). The plan's summary is the release record, so a Linear-side exception (a feature not `QA passed`, a Triage blocker) goes in that line too; with a red workflow-side part (scenarios, a nightly) and no `known_issues` the plan's `Summary` job fails and the apply refuses the version. Artem starts it (`workflows.md` § Commands). When he says it ran (the run `Release plan · <version>`), read its outcome and logs (`workflows.md` § Outputs). The plan passed when its `Summary` job passed; the run as a whole can show red on a failed nightly that a known issue covers. Then: Core's prod diff in its two Core runs, each Surfaces family's in its job's log, against what the scope predicts; the console has no dry run in the plan (the apply checks it on a channel first), so say it was not previewed. The gate table and known issues are in the run's summary, on the run page only: ask Artem to read them there, or say they were not read. Anything the scope does not explain (a family changing that should not, a migration nobody expected) is said plainly, and the release waits on Artem's call.
5. **Hand Artem the apply**, the same way, after he has read the plan. Artem starts it. When he says it ran (the run `Release apply · <version>`), read its outcome: every family job green, `Public hosts` green, `Tag` green and `prod/v<version>` on both repositories (`workflows.md` § Outputs). A red apply is reported job by job: which families moved, what prod runs now. The apply follows the latest completed plan of the version; if it refuses, say whether that plan's `Summary` failed. After a cancel, check Core's `Deploy ... → prod · <version>` runs, which a cancel does not stop. Never re-run it or any job of it; a rerun is Artem's, like the start. A `prod/v` tag is never moved, so a second apply of an already tagged version deploys and then fails at its tag. The first real sign-in on prod is Artem's (there is no prod test account).
6. **After a green apply**: move the scope's `FEAT` features to `Released`, and draft the team post. `references/after-apply.md`.

## Rules

- **Only Artem starts plan and apply.** The agent hands links and commands and never runs them, even if its `gh` login could. It also never runs `gh run rerun` or `gh run cancel` on a plan, apply or prod deploy run (Core's dispatched runs included), nor `gh workflow run deploy.yml`: a rerun is handed to Artem like the start. The workflows refuse every account but the release owner (Artem's, set in the repositories' `RELEASE_OWNER` variable), in every prod job, but they cannot tell Artem from an agent using his login; this rule is what holds.
- **No cloud call by any route.** Everything that touches prod runs in the workflows. No deployer shell, no `gcloud`, no desk deploy as a fallback. If a workflow is missing or cannot reach prod (it stops in its first job until `RELEASE_OWNER`, prod's federation variables and Core's prod target exist, `workflows.md` § Preconditions), stop and report which.
- **The agent sends nothing.** The team post is a draft for Artem.
- **Tracker writes are limited to `FEAT` `Released`**, after a green apply, on Artem's yes to the list. No `ENG` status moves: on 2.0 they are Linear's GitHub integration's and QA's (`events-tracker-base` § Statuses → ENG flow on 2.0). No mark is set by hand. Linear Releases are not used.

## References

- `references/gate.md`: reading the scope and each part of the gate, blockers, the known-issue record.
- `references/after-apply.md`: the `Released` moves and the team post draft.
- `../../references/workflows.md`: the workflows by role, their names, inputs, jobs and outputs, the preconditions, the builds CI cannot carry, the commands.
