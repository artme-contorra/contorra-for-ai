---
name: cutting-a-test-build
tools:
  - Read
  - Bash
  - mcp__linear-server__list_issues
  - mcp__linear-server__get_issue
  - mcp__linear-server__list_comments
description: >
  Cut a test build of Events 2.0: decide whether there is enough to build and
  what the build is for, start the test-build CI workflow, wait for it, read
  its result, and hand Artem a draft QA post for `#team-events-ru`. The
  workflow composes `main` of Core and Surfaces, numbers the build
  `2.<stage>.<build>`, tags both repositories, deploys to the test stand and
  runs the stand scenarios; the skill holds the judgement around it. Use
  whenever someone wants a new build on the test stand, asks whether a build
  is due, wants QA to get the latest fixes, or asks what went into the last
  test build. Never touches prod, sends nothing, moves no tracker status.
  Trigger phrases: "собери тестовую сборку", "выкати на тест", "нужна новая
  сборка", "пора собирать?", "cut a test build", "new build for QA",
  "what's in the test build".
---

A test build is cut deliberately, not on every merge: the team merges many PRs a day, and the stand has to hold still for QA. One dispatch of the test-build workflow cuts one build. Everything a script can do happens in that workflow; this skill decides whether to start it, reads what it reports and writes the QA post draft. The workflow's name, inputs and outputs are in `../../references/workflows.md` (the plugin's `references/`; the one place they are named).

The tracker follows the code by itself: Linear's GitHub integration has already moved every merged 2.0 issue to `ENG` `QA` (`events-tracker-base` § Statuses → ENG flow on 2.0). This skill moves no status.

## Steps

1. **Decide that a build is due.** Read what merged on `main` of Core and Surfaces since the last `v2.*` tag and the `ENG` issues those PRs name. Say in two or three lines what the build would be for (a fix QA is waiting on, a feature that opens, a batch of fixes) and whether that is enough. A build nobody will check is not due. Detail: `references/procedure.md` § 1.
2. **Check what would stop it.** The workflow stops when Core and Surfaces pin different Proto commits, when a `floor` is not green (read on the head of the PR that made `main`'s tip, since `main` itself carries none), or when neither `main` moved. Read them first, so a run is not wasted; if one fails, say which and stop (`procedure.md` § 2). A new stage (`2.<n+1>.0`) only on Artem's word. Changing the opened set (`release.json`) is a PR decided before the build, never part of it.
3. **Start the workflow and wait.** If the previous test build ended untagged, first classify its failure: on the machine (network, runner, registry, a cancel after a queue timeout) or in the code (a check that would fail again on the same commits; `procedure.md` § 4). A code failure is never retried: it needs a fix PR. A machine failure, while neither `main` moved, is retried once with the same inputs (same number, the images it pushed). Once a `main` moved and an image of that version was pushed, the version is spent: pass it (and any earlier spent ones) as `skipped`. Dispatch from `main` of Surfaces and watch the run to the end (`procedure.md` § 3). Tell the person the run's link when it starts. No notice to QA before the dispatch: QA hears about a build from Artem.
4. **Read the result.** Green: the version, the commits, the scenario verdict, from the tag's manifest (a run's summary is not readable with `gh run view`). Red: the job that failed and the first failing line, and what it means; whether the version was tagged (scenarios red after tagging) or left untagged (`procedure.md` § 3 decides retry or `skipped`). A red family job may already have replaced its service: read its log, say which families moved and what test runs now, and draft a stand-status line for Artem (`procedure.md` § 4, `qa-post.md`). Never relax a test or a limit to get a green build; a fix is a PR, and the next build gets the next number.
5. **Draft the QA post** for `#team-events-ru` from the manifest: the version, what to check, the `ENG` issues now in `QA` that this build carries, known problems. Format: `references/qa-post.md`. Hand the draft to Artem; he edits and sends it.

## Rules

- **Test only.** This skill never starts, reruns or cancels plan or apply, never names a prod host as a target, and makes no cloud call by any route. Prod is the `releasing` skill's, and only Artem starts it.
- **The agent sends nothing.** The QA post and any stand-status line are drafts for Artem; QA is told about a build by Artem. No Slack, no Linear comment, no message to QA.
- **No tracker writes.** `ENG` statuses on 2.0 are the integration's and QA's. A merged issue that is not in `QA` is reported, not moved. `FEAT` statuses are not this skill's either (`procedure.md` § 5).
- **No fallback to a laptop.** If the workflow is missing, refuses, or cannot reach test, stop and report. Do not run the per-family deploy scripts by hand in its place.

## References

- `references/procedure.md`: the reads, the dispatch and the wait, how to read a red run, what the build does not do.
- `references/qa-post.md`: the QA post draft and the stand-status line, their parts and their language.
- `../../references/workflows.md`: the workflows by role, their names, inputs, jobs and outputs, the commands.
