# Triage labels

The `triage` skill works in **canonical role names**. This file maps each role to the label
string actually used on this repo's tracker.

This repo uses the **defaults** — every label string equals its role name.

## Category roles

Every triaged issue carries exactly one.

| Role | Label | Meaning |
|---|---|---|
| `bug` | `bug` | Something is broken |
| `enhancement` | `enhancement` | New feature or improvement |

## State roles

Every triaged issue carries exactly one. If two ever conflict, flag it and ask the maintainer
before doing anything else.

| Role | Label | Meaning |
|---|---|---|
| `needs-triage` | `needs-triage` | Maintainer needs to evaluate |
| `needs-info` | `needs-info` | Waiting on reporter for more information |
| `ready-for-agent` | `ready-for-agent` | Fully specified, ready for an AFK agent |
| `ready-for-human` | `ready-for-human` | Needs human implementation |
| `wontfix` | `wontfix` | Will not be actioned |

## Transitions

An unlabelled issue normally enters at `needs-triage`, then moves to `needs-info`,
`ready-for-agent`, `ready-for-human`, or `wontfix`. `needs-info` returns to `needs-triage` once
the reporter replies. The maintainer may override at any time — flag unusual transitions and ask
first.

## Notes

- `to-spec` applies **`ready-for-agent`** to the spec it publishes. No further triage is needed
  on a spec, and tickets produced by `to-tickets` are already agent-ready — **do not triage
  them**. Triage is only for issues you did not create.
- The `wayfinder:*` labels are a **separate axis** and unrelated to triage. A wayfinder ticket is
  a decision to resolve, not a request to evaluate; it takes no triage label.
