# 0003. Blobs hold native bytes only

- **Status:** accepted
- **Date:** 2026-08-16
- **Settled by:** [#11](https://github.com/seanrobertwright/session-backpack/issues/11), [#12](https://github.com/seanrobertwright/session-backpack/issues/12) · **Spec:** [v1 §3.1, §5.4](../spec/v1.md)

## Context

Backpack must both **restore** sessions into nine evolving native formats and **display** them in one
normalized reader. The obvious move is to store both: the native bytes for restore, and a parsed
projection for fast browsing. Nine tools' formats churn constantly, and our parsers will be wrong at
first.

## Decision

**Blobs hold exactly what the tool wrote, byte for byte. No normalized projection is ever persisted in
the vault.**

The projection is derived at **read time** from the stored native bytes and cached in the per-machine
SQLite index, keyed by `(blob_hash, adapter_version)`. The manifest carries only lightweight **listing**
metadata — title, turn count, time range, size, degradation counts — folded from that same projection at
capture time by the same `project()` call.

## Consequences

**Makes easy.** Restore is byte-exact by construction. **A parser fix in a later release retroactively
improves every session ever captured**, because truth was preserved rather than a rendering of it — which
is what lets the projection be a faithful superset (spec §5.4) and lets the turn vocabulary evolve freely
under `adapter_version`. One artifact to version instead of two. Browsing never decrypts a transcript,
because listing data rides in the manifest.

**Makes hard.** Version skew is real: a machine on an older build may lack the parser for a session a
newer machine captured, and that session degrades to *captured by a newer Backpack* rather than rendering.
Stored projections would have covered this — at roughly double the vault size, a second schema to version
and migrate, and the loss of retroactive parser fixes.

**Forecloses.** Any capture-time normalization, including trimming a torn record to the last valid line:
a trimmed blob is no longer native bytes (ADR 0006). Also `VACUUM INTO` for SQLite-backed tools, whose
output is semantically equal but not byte-equal.

**Second-order.** Because `meta` is folded from the projection at capture, an all-or-nothing parser would
write `title: null` into the vault permanently. This is why degradation is **per-turn** rather than
per-file.
