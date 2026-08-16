# 0009. State enters the vault only if another machine needs to know it

- **Status:** accepted
- **Date:** 2026-08-16
- **Settled by:** [#13](https://github.com/seanrobertwright/session-backpack/issues/13) · **Spec:** [v1 §2.3](../spec/v1.md)

## Context

Backpack accumulates state of many kinds: a SQLite index, a projection cache, quarantine flags,
compaction candidates, UI preferences, capture settings, machine labels, retention policy. Each arrives
with its own local argument for going in the vault — *it would be nice if my other machine knew this
too.*

But the vault is append-only, shared, and lives in the user's own cloud storage. Anything put there is
there permanently, propagates to every machine, and becomes a compatibility surface. Deciding each case
on its merits would produce a boundary nobody could predict or defend.

## Decision

> **State goes in the vault if and only if another machine needs to know it.**

| State | Where |
|---|---|
| Machine id | Local |
| Machine label / os / status | **Vault** |
| SQLite index + projection cache | Local |
| Quarantine flags | Local |
| Compaction candidate set | Local |
| UI preferences (view shape, skin) | Local |
| Capture cadence, which installs to watch | Local |

Where a setting genuinely governs *shared* content, the vault-scoped slot `config/<ts>.mf.age` exists
(write-once, newest wins) — deliberately **not** in `vault.json`, which is the only mutable file in the
vault and therefore the only routine conflict candidate. **v1 puts nothing in it.**

## Consequences

**Makes easy.** Every subsequent question answers itself without a special case. Two rules that would
otherwise need arguing follow directly: **one machine's suspicion must never propagate as fact** — which
is why a failed integrity verification is local quarantine rather than a vault mutation, since a
partially-synced blob is byte-for-byte indistinguishable from a corrupt one — and the compaction
candidate set is likewise local.

**Makes hard.** A fresh install starts from default UI preferences. There is no "adopt my desktop's
setup", and buying that one-time convenience would mean parking presentation state in the shared
append-only vault permanently.

**Forecloses.** Cross-machine UI preference sync (out of scope, spec §12). Also any vault-scoped
retention policy — the `config/` slot was created for exactly that, and ADR 0010 then removed the need
for it, because compaction has no policy to diverge on.
