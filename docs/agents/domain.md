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

`CONTEXT.md` **does not exist yet** — this is a greenfield repo whose design currently lives in
the closed decision tickets of
[Wayfinder map: Backpack v1 spec #1](https://github.com/seanrobertwright/session-backpack/issues/1).

Seeding it is part of
[Assemble the v1 spec #17](https://github.com/seanrobertwright/session-backpack/issues/17), which
inherits the session / branch / capture vocabulary from
[#13](https://github.com/seanrobertwright/session-backpack/issues/13). This matters more than it
looks: every `implement` session starts with **cleared context** against this repo, so until the
glossary is in-tree each one would have to re-read the tracker to learn the language.
