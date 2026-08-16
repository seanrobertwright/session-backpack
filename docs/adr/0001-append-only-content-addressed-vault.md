# 0001. The vault is append-only and content-addressed

- **Status:** accepted
- **Date:** 2026-08-16
- **Settled by:** [#11](https://github.com/seanrobertwright/session-backpack/issues/11) · **Spec:** [v1 §3.1](../spec/v1.md)

## Context

The vault lives inside a folder the user already syncs — OneDrive, Dropbox, Google Drive, Syncthing, a
git repo. Backpack operates no sync engine of its own and cannot influence ordering, conflict handling
or propagation delay. Whatever we write must be safe under a sync engine that may deliver files late,
out of order, or twice, and that resolves its own races by leaving conflicted copies behind.

Any design containing files that are routinely **rewritten** makes sync conflicts a routine event.

## Decision

The vault is a **log-structured, content-addressed, append-only archive.**

- Every captured file becomes an immutable **blob** at `blobs/<hh>/<blake3-hex>.age`, named by the
  **BLAKE3 hash of its plaintext** — taken before encryption, because age is non-deterministic and
  hashing ciphertext would defeat deduplication entirely.
- Every backup writes one **write-once manifest** per (session, capture), stored in that session's own
  directory.
- **Nothing in the vault is ever rewritten.** `vault.json` is the sole exception, and it changes only on
  a format bump or a recipient change.
- Blobs are written **before** the manifest that references them, always.

## Consequences

**Makes easy.** Concurrent writes from any number of machines cannot collide, so no lock, no leader and
no coordination protocol is needed anywhere in the system. Full history comes for free, as does
deduplication across captures and across machines. Integrity verification is free, because the filename
*is* the checksum. Two machines writing the same blob name with different ciphertext is harmless — both
decrypt to identical bytes.

**Makes hard.** Vault growth is O(size × captures) for whole-file blobs; the countermeasures are capture
cadence (spec §4.1) and compaction (ADR 0010), not the format. The vault is thousands of small files,
so enumeration needs a local index rather than a directory walk.

**Forecloses.** Any operation that edits history in place. A mistake, once captured, is permanent — which
is why capture is an allow-list rather than a deny-list root (ADR 0005): a newly-captured secret would be
unfixable.
