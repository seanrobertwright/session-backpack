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
a future change would have to fight. That promotion was carried out in
[Assemble the v1 spec #17](https://github.com/seanrobertwright/session-backpack/issues/17), which
selected ten:

| ADR | Title | Spec |
|---|---|---|
| [0001](./0001-append-only-content-addressed-vault.md) | The vault is append-only and content-addressed | §3.1 |
| [0002](./0002-vault-layout-and-identity.md) | Vault layout: readable tools, opaque sessions | §3.2, §5.5, §5.8 |
| [0003](./0003-blobs-hold-native-bytes-only.md) | Blobs hold native bytes only | §3.1, §5.4 |
| [0004](./0004-two-number-format-versioning.md) | Two-number format versioning with per-session degradation | §3.4 |
| [0005](./0005-adapters-are-untrusted.md) | Adapters are untrusted; the core owns every byte | §5.1, §5.2 |
| [0006](./0006-never-rewrite-file-contents.md) | Backpack never rewrites file contents | §4.2, §5.6, §7.2 |
| [0007](./0007-per-vault-keypair.md) | Per-vault X25519 keypair, recoverable with the age CLI | §9.1 |
| [0008](./0008-machine-is-one-install.md) | A machine is one install of Backpack | §7.1 |
| [0009](./0009-local-vs-synced-boundary.md) | State enters the vault only if another machine needs it | §2.3 |
| [0010](./0010-deletion-never-happens-on-a-guess.md) | Deletion never happens on a guess | §8 |

Everything else the map decided is spec content and lives in
[`docs/spec/v1.md`](../spec/v1.md); its Appendix A indexes all eighteen closed tickets.
