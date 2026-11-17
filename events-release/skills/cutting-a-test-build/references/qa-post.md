# The QA post draft

Loaded from `cutting-a-test-build` → `SKILL.md` step 5. The post goes to `#team-events-ru`, in Russian,
because that is the team's channel for process on the test environment. The agent writes a draft and
hands it to Artem in chat; Artem edits and sends it. The agent never posts it.

If the host workspace has a skill for writing in Artem's voice, follow it for the wording. Either way:
short sentences, no filler, no long dashes.

## Parts, in this order

1. **The version and the stand.** One line: build `2.<stage>.<build>` is on the test stand (name the
   hosts QA uses), from when.
2. **What to check.** The `ENG` issues this build carries that are now in `QA`, one line each:
   `ENG-N`, what changed in plain words, the `FEAT-N` it relates to. Group by feature when several
   issues touch one.
3. **Features to look at again** (if any): features that were `QA failed` and whose fix this build
   carries. Name them in the post so QA rechecks them. Their move back to `Ready for QA` is not this
   skill's (`procedure.md` § 5): raise it with Artem in chat, beside the draft.
4. **Newly opened features** (if `release.json` changed since the last build): `FEAT-N` and its title.
5. **Known problems**: what is broken on this build and already filed, with its `ENG-N`, so QA does
   not file it again.
6. **The ask**: name the version in every bug report and comment about this build.

Leave out: commit hashes, image digests, workflow names, scenario counts. They are in the manifest for
whoever needs them.

## Example skeleton

```
Новая сборка на тестовом стенде: 2.2.4 (admin-test, mcp-test), с 14:20.

Что проверить:
- ENG-231: <что изменилось> (FEAT-3)
- ENG-235: <что изменилось> (FEAT-167)

Известные проблемы: ENG-240 <коротко>, уже в работе.

В баг-репортах по этой сборке указывайте версию 2.2.4.
```
