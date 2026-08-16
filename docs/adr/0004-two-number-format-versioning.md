# 0004. Two-number format versioning with per-session degradation

- **Status:** accepted
- **Date:** 2026-08-16
- **Settled by:** [#11](https://github.com/seanrobertwright/session-backpack/issues/11), [#15](https://github.com/seanrobertwright/session-backpack/issues/15) · **Spec:** [v1 §3.4](../spec/v1.md)

## Context

Updates **prompt before installing** and are delivered per machine, so a user who defers on their laptop
and accepts on their desktop has two Backpack versions reading and writing **one shared vault**. Since
the product's premise is a single vault shared across machines via BYO sync, **this is the main case, not
an edge case.**

Updating machine A must never strand machine B.

## Decision

`vault.json` carries **`format_version`** (bumped for additive change) and **`min_reader_version`**
(bumped only for a breaking change). Every manifest carries its own `v`.

| Situation | Behaviour |
|---|---|
| Old reader, additive change | Works. Unknown fields ignored. |
| Old reader meets a newer manifest | **That session only** degrades. The rest of the vault is fully usable. |
| Build older than `min_reader_version` | Refuse the vault, prompt to update. |
| Any build **writing** | Always safe — append-only, never mutates. |
| **Compactor meets an unparseable manifest** | **Refuse to run.** |

Unknown enum values deserialize to an `Unknown` variant rather than failing.

## Consequences

**Makes easy.** Additive format evolution without a migration: `segments[]` for delta storage,
`supersedes` for prefix proofs, new `meta` fields, new `FileRole` variants. Old machines keep backing up
safely forever, which matters most for silent daemons whose owner is least likely to notice a lockout.

**Makes hard.** A newer machine's sessions may be unreadable on an older one until it updates. Accepted:
the condition is transient and per-session, and the bytes are intact throughout.

**Forecloses.** A single strict version — an additive bump would lock every un-updated machine out of its
own backups. A fully permissive scheme — changed semantics would be silently misread, and the compactor
would get no signal to hold back, which is precisely where the destructive case lives.

**Note on the last row.** Under ADR 0010's proof-based compaction, an unparseable manifest can only cause
a *missed proof* — a failure to delete, which is safe. The refusal rule is therefore now
defence-in-depth rather than load-bearing. It is kept anyway: compaction is never urgent, so
belt-over-braces on the only destructive operation in the system costs nothing.
