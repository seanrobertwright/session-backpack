# Issue tracker

Issues for this repo live in **GitHub Issues** on `seanrobertwright/session-backpack`, accessed
with the [`gh`](https://cli.github.com/) CLI.

Skills that read from and write to this tracker: `to-spec`, `to-tickets`, `triage`, `qa`, `wayfinder`.

## Operations

| Intent | Command |
|---|---|
| Create an issue | `gh issue create --title "..." --body-file <file>` |
| Read an issue | `gh issue view <n> --json number,title,body,labels,state,assignees` |
| Read with comments | `gh issue view <n> --comments` |
| List open issues | `gh issue list --state open --json number,title,labels` |
| Comment | `gh issue comment <n> --body-file <file>` |
| Label | `gh issue edit <n> --add-label "<label>" --remove-label "<label>"` |
| Claim | `gh issue edit <n> --add-assignee <user>` |
| Close | `gh issue close <n> --reason completed` |

Prefer `--body-file` over `--body` for anything multi-line; it avoids shell quoting damage on
Windows.

## PRs as a request surface

**Off.** External pull requests are *not* part of the triage queue — `triage` covers issues only.
Flip this flag if you later want incoming PRs triaged like issues, and define who counts as
"external" when you do.

## Wayfinding operations

`wayfinder` maps live on this tracker too. Its tracker-specific expressions:

| Wayfinder concept | GitHub expression |
|---|---|
| Map | An issue labelled `wayfinder:map` |
| Ticket | A **sub-issue** of the map issue |
| Ticket type | A `wayfinder:<type>` label — `research`, `prototype`, `grilling`, `task` |
| Claim | Assign the issue to the dev driving the map |
| Blocking | GitHub's **native** blocked-by relationship |
| Frontier | Open sub-issues that are unblocked and unassigned |

Sub-issues and blocking relationships are not exposed by `gh issue` flags; reach them through
the GraphQL API:

```bash
# children of a map
gh api graphql -f query='{repository(owner:"seanrobertwright",name:"session-backpack"){
  issue(number:1){subIssues(first:50){nodes{number title state}}}}}'
```

The current map is [Wayfinder map: Backpack v1 spec #1](https://github.com/seanrobertwright/session-backpack/issues/1).
