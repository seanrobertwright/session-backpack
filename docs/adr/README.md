# Architecture Decision Records

One file per hard-to-reverse decision, named `NNNN-kebab-case-title.md` with a monotonic number.

An ADR records a decision that would be expensive to undo — a format constant, a trust boundary,
a dependency you cannot easily shed. Reversible choices do not need one.

## Template

```markdown
# NNNN. <Title>

- **Status:** proposed | accepted | superseded by [NNNN](./NNNN-....md)
- **Date:** YYYY-MM-DD

## Context

The forces at play, and why a decision is needed now.

## Decision

What was decided, stated in the active voice.

## Consequences

What this makes easy, what it makes hard, and what it forecloses.
```

## Relationship to the wayfinder map

The v1 design was settled as decision tickets on
[Wayfinder map: Backpack v1 spec #1](https://github.com/seanrobertwright/session-backpack/issues/1),
not as ADRs. Those resolutions are the record for everything decided before the first line of
code.

Not all of them warrant an ADR — most are spec content and belong in the v1 spec document. The
ones that do are the **permanent format constants and trust boundaries**, because those are what
a future change would have to fight. Deciding which of the closed tickets get promoted to ADRs is
part of [Assemble the v1 spec #17](https://github.com/seanrobertwright/session-backpack/issues/17).
