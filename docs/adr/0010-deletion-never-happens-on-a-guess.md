# 0010. Deletion never happens on a guess

- **Status:** accepted
- **Date:** 2026-08-16
- **Settled by:** [#13](https://github.com/seanrobertwright/session-backpack/issues/13), [#19](https://github.com/seanrobertwright/session-backpack/issues/19) · **Spec:** [v1 §8](../spec/v1.md)

## Context

Every other operation in Backpack is safe by construction: the vault is append-only, so writing cannot
destroy anything (ADR 0001). **Deletion is the sole exception, and therefore the only place the system
can lose a user's data.**

It is also where the eventually-consistent substrate bites hardest. A classic mark-and-sweep collector
builds a referenced set from the manifests it can currently see. But machine B writes blob `X`, then
manifest `M` referencing it, and the sync engine propagates the two **independently and in no guaranteed
order** — so machine A can see `X` without `M`, conclude it is garbage, and delete live data. The machine
that captured it did nothing wrong. Nothing else in the design catches this: verify-on-read finds the
damage far too late.

Meanwhile the vault grows at ~6.3 GiB/yr/tool/machine and OneDrive's free tier is 5 GB, so doing nothing
is not indefinitely viable either.

## Decision

> **Deletion never happens on a guess.**

**v1 compacts; it does not collect.** Two operations were hiding under one name and are now separated:

- **Compaction** deletes blobs carrying *no unique information*. Its question — *do these bytes survive
  inside a blob that still exists?* — is a **fact**.
- **Retention** deliberately forgets history the user might still want. Its question — *does the user
  still want this?* — is **unknowable**. v1 has none, and none is planned (spec §12).

The invariant that replaces mark-and-sweep outright:

> **A blob may be deleted iff a surviving proof chain reconstructs it from a blob that still exists.**

- The **proof is recorded forward by the core at capture time**, while the new bytes are legitimately in
  hand: hash the first `L` bytes and check against the predecessor's hash, then write
  `supersedes: { blob, len }` on the **successor** manifest (manifests are write-once, so it cannot be
  stamped on the predecessor). No old blob is read; nothing hydrates.
- **Manifests are never deleted. Only blobs are.**
- Re-verified in a **single streaming pass** before deleting, and only once survivor and proof are
  **7 days** old.
- Runs from **any machine, no lock, no designated collector**; a weekly idle sweep over the machine's own
  shard.

Three other rules are the same invariant in its other locations: the compactor refuses to run on an
unparseable manifest; the conflicted-copy reconciler will not delete a copy whose manifest it cannot
parse; and `Collision::Replace` first captures what it supersedes.

## Consequences

**Makes easy.** The sync race **evaporates** — the collector never deletes on *absence of evidence*, only
on *presence of a proof*, and the proof is itself the recovery path. There is no referenced set to
compute and therefore no way to compute it wrong. Failure is safe by default: a rewritten file, a
`/compact`, a rotated log or a torn tail simply **stops being compactable** rather than becoming data
loss. Reference counting is irrelevant. Compaction is **lossless**, so it needs no setting, no prompt and
no UI at all.

**Makes hard.** One new legitimate state for verify-on-read — **absent by design**, distinct from
*missing* — and reading a compacted capture costs a chain walk plus a truncation. A retired machine's
shard, and your own shard from before a reinstall, are never compacted and are kept forever.

**Forecloses.** Any user-facing ability to forget. And it imposes a **hard v1 requirement**: the reader
must understand proof-chain reconstruction **from day one**, even though the compactor will rarely have
run. `min_reader_version` cannot patch this later, because the manifest an old build chokes on is the
*superseded* one — unchanged and perfectly parseable — while the proof lives on the successor. **A
collector shipped ahead of reader support would make old builds report intact sessions as corrupt.**
