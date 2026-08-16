# Domain docs

This repo is **single-context**: one domain, one glossary, one ADR directory.

| Doc | Location | Holds |
|---|---|---|
| Glossary | `CONTEXT.md` (repo root) | The ubiquitous language — every domain term, defined once |
| ADRs | `docs/adr/` | Hard-to-reverse decisions, one file per decision |

There is no `CONTEXT-MAP.md`; that is the multi-context layout and does not apply here.

## Consumer rules

Skills that touch the domain (`to-spec`, `to-tickets`, `implement`, `tdd`, `triage`,
`grill-with-docs`, `domain-modeling`, `code-review`) must:

1. **Read `CONTEXT.md` before writing anything that names a domain concept**, and use its terms
   verbatim. If a needed term is missing or fuzzy, that is a signal to sharpen it via
   `domain-modeling` — not to invent a synonym.
2. **Read any ADR in the area being touched** before proposing a design, and respect it. An ADR
   is overturned by a new ADR that supersedes it, never by silent divergence.
3. **Never let two words do one job.** If a term is doing three jobs, split it and record the
   split in `CONTEXT.md`.

## Current state

Seeded. [`CONTEXT.md`](../../CONTEXT.md) holds the glossary and
[`docs/adr/`](../adr/) holds ten ADRs, both written by
[Assemble the v1 spec #17](https://github.com/seanrobertwright/session-backpack/issues/17) alongside
[`docs/spec/v1.md`](../spec/v1.md) — the design reference folded from the eighteen closed decision
tickets of
[Wayfinder map: Backpack v1 spec #1](https://github.com/seanrobertwright/session-backpack/issues/1).

**Read `CONTEXT.md` before the spec.** Every `implement` session starts with cleared context against
this repo, and the glossary is what stops each one having to re-read the tracker to learn the
language. Its closing section — *words we deliberately do not use* — is load-bearing, not
decoration: "GC" and "retention" both name mechanisms this design explicitly rejected.

Ticket resolutions on the map remain the record of **why**; the spec is the record of **what**. Where
the two disagree, the spec is current.
