# Reading the Events tracker

Read-only recipes over `ENG` (Events Engineering), `DES` (Events Design) and
`REQ` (Feature requests, where parked requests live). Always pass `team:` so
queries don't pull workspace-wide. Report status in the buckets from SKILL.md
(Not started / In progress / Done / Won't ship, plus Parked on `REQ`) — not the
team's internal status names — unless the person asks for detail.

## 1. Status of a specific issue

Given an identifier like `ENG-42`, `DES-7` or `REQ-3`:

```
get_issue id:"ENG-42"
```

Report: the title, which bucket it's in, who it's assigned to, and the most
recent update (pull recent comments with `list_comments issueId:"ENG-42"` if a
fuller picture helps).

## 2. What's happening in Events right now

Active work across both teams:

```
list_issues team:"Events Engineering" state:"started"
list_issues team:"Events Design" state:"started"
```

Here `state:"started"` is a Linear status-*type* filter that returns whatever is
actively being worked — i.e. the **In progress** bucket. Summarize by project
where possible, and report it as "in progress" rather than by internal status
name. For a narrative of recent movement, add project status updates (recipe 4)
for the active projects.

## 3. Find an issue by topic

```
list_issues team:"Events Engineering" query:"<topic words>"
list_issues team:"Events Design" query:"<topic words>"
list_issues team:"Feature requests" query:"<topic words>"
```

`query` searches title + description. Widen or narrow the words if the first
pass misses. A request the team is not working on is found in `REQ`, not `ENG`;
report it as parked and quote the revisit condition from its comments
(`list_comments issueId:"REQ-N"`).

## 4. Summarize a project

```
list_projects team:"Events Engineering"        # find the project by name
get_project id:"<project id or name>"
list_issues project:"<project name>"           # its issues, group by bucket
get_status_updates projectId:"<project id>"     # latest narrative updates
```

Give: what the project delivers (its description), how its issues split across
the four buckets, the lead, and the latest status update.

## 5. Exec overview (initiatives → projects)

```
list_initiatives                                # strategic banners
list_projects team:"Events Engineering"
list_projects team:"Events Design"
```

Roll up: each initiative, the projects under it, each project's coarse status
and lead. This is the "state of the product" view.
