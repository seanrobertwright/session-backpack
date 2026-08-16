# 0007. Per-vault X25519 keypair, recoverable with the age CLI

- **Status:** accepted
- **Date:** 2026-08-16
- **Settled by:** [#14](https://github.com/seanrobertwright/session-backpack/issues/14) · **Spec:** [v1 §9](../spec/v1.md)

## Context

Two requirements pull hard against each other. **Backups must run unattended forever** — the watcher
fires 90 seconds after a session goes idle, on every machine, across reboots, with no user present. And
**the vault must be encrypted at rest**, because transcripts contain source code, pasted credentials and
client work.

Direct passphrase encryption of every file satisfies the second and destroys the first: every capture
would need the passphrase in memory, so either the app prompts at launch or it caches a passphrase
indefinitely.

A third requirement shapes the answer: **the archive must outlive Backpack.** A backup tool whose data
is unreadable without the tool has moved the risk rather than removed it.

## Decision

**A per-vault X25519 keypair.**

- The vault's **public key lives in plaintext** in `vault.json`, so **any machine encrypts new backups
  unattended, forever**, with nothing unlocked.
- The **private identity** lives in the vault at `identity.age`, wrapped under the user's passphrase with
  age's scrypt passphrase encryption (library defaults, no custom KDF tuning).
- Unlock is **lazy** — on first protected action, never at launch, never for a backup — with
  **"Remember on this device" checked by default**, caching the **decrypted identity** (never the
  passphrase) in the OS keychain.
- **A recovery kit is generated at vault creation and every file is encrypted to both recipients from
  byte one.** Shown once, never stored by Backpack.

The vault opens with the standalone `age` / `rage` CLI: `age -d -i identity.key`.

## Consequences

**Makes easy.** One passphrase entry per machine, ever — and none at all for a user who runs Backpack as
a silent backup daemon. Encryption is never a reason a backup did not happen. The archive is readable in
twenty years by anyone with the age CLI, which is what justifies the readable tool namespace and
per-session manifests in ADR 0002.

**Makes hard.** `id_salt` must be plaintext so unattended capture can compute opaque ids without
unlocking — the residual privacy cost recorded in ADR 0002. A cached identity in the OS keychain is a
real key at rest on each machine, protected by the OS user account.

**Forecloses.** Retro-fitting a recovery recipient: adding one later could not protect files already
written, which is why it happens at creation or not at all. Full key rotation is out of scope for v1 —
passphrase change is a **re-wrap only**, and the UI says so honestly rather than implying that old
backups are re-protected.

**Stated plainly at creation:** lost passphrase + lost recovery key + no cached machine = lost vault.
