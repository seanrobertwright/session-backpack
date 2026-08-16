# 0002. Vault layout: readable tool namespace, opaque session identity

- **Status:** accepted
- **Date:** 2026-08-16
- **Settled by:** [#11](https://github.com/seanrobertwright/session-backpack/issues/11), [#12](https://github.com/seanrobertwright/session-backpack/issues/12) · **Spec:** [v1 §3.2, §5.5, §5.8](../spec/v1.md)

## Context

age encrypts contents but **never filenames**. Every path component in the vault is visible to anyone who
can list the synced folder, the sync provider included. Meanwhile the vault is append-only (ADR 0001), so
a path component chosen now can never be renamed later — there is no migration that rewrites history.

Two opposing pulls: fully-opaque paths maximise privacy but degrade bare-`age` recovery to decrypting
thousands of hashes one by one; fully-readable paths make recovery trivial but publish the user's project
paths and client names to a cloud provider.

## Decision

```
sessions/<tool>/<opaque-id>/<machine>/<ts>.mf.age
state/<tool>/<machine>/<ts>.mf.age
machines/<machine-id>/<ts>.mf.age

opaque-id = blake3(id_salt || tool || native_id), 128-bit hex
```

- **`<tool>` is readable** and equals the adapter's `descriptor().id`, which is therefore a **permanent
  format constant.** One adapter per vault namespace — pi and oh-my-pi are separate adapters over a
  shared module, never one merged id.
- **`<opaque-id>` hides the session's native identifier**, and with it any project path baked into it.
- **`native_id` must always be the tool's own session-level identifier, never a path-derived value.**
- **Project and workspace are encrypted manifest content**, never a path component.

## Consequences

**Makes easy.** Bare-`age` recovery: a human can `cd sessions/claude-code/<id>/<machine>`, decrypt the
newest manifest, and read a file list. Adapter dispatch, restore targeting and compaction scoping are all
one path component away.

**Makes hard.** `id_salt` must be **plaintext**, because unattended backup computes opaque ids without
unlocking the vault (ADR 0007). Opaque ids are therefore preimage-resistant but **not resistant to a
confirmed guess** — someone with folder access who guesses a `native_id` can test whether that session is
present. The no-path-derived-ids rule eliminates rather than narrows this: every candidate case (Gemini's
`sha256(abs path)`, Copilot's `md5(path+ctime)`) identifies the *project*, not the session, and both tools
expose a high-entropy session id.

**Forecloses.** Renaming an adapter id after v1 — it would orphan every session captured under the old
one, permanently. Adding a `<project-hash>` path component, which would put a hash of an absolute path
back on the visible surface.

**Leaks, accepted and stated:** which tools you use, roughly how many sessions, how many machines, when
you worked.
