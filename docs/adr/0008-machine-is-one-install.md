# 0008. A machine is one install of Backpack

- **Status:** accepted
- **Date:** 2026-08-16
- **Settled by:** [#13](https://github.com/seanrobertwright/session-backpack/issues/13) · **Spec:** [v1 §7.1](../spec/v1.md)

## Context

`<machine>` is a **plaintext path component** in the vault layout (ADR 0002), and per-machine sharding is
what makes concurrent captures from several machines physically unable to collide. Because the vault is
append-only, whatever a machine is called in a path can never be renamed.

So the identifier must be stable for the life of an install, unique across installs, cheap to mint, and
safe to publish to a cloud provider.

## Decision

**A machine is one install of Backpack**, identified by a **128-bit random opaque id** minted at first
run and stored in **machine-local config — never in the vault.**

Label, OS and `status` (`active` | `retired`) live in append-only records at
`machines/<machine-id>/<ts>.mf.age`, written on **first run, label change and explicit retirement only**.
Manifests additionally carry `machine: { id, label, os }` redundantly, so a human decrypting one manifest
with the bare `age` CLI can see who wrote it without consulting the registry.

## Consequences

**Makes easy.** Renaming a machine is free — the label touches no path, which matters because an
append-only vault cannot rewrite one. Retirement is expressible. A machine that has never captured
anything still exists. The answer to *which machine am I?* comes from a file the app itself wrote.

**Makes hard.** Reinstalling Backpack mints a **new** machine; the old shard stays in the vault and
nothing writes to it again. This is honest rather than harmful — the UI presents it as
*`sean-desktop` (retired)* — but the orphaned shard is never compacted (ADR 0010), so it costs bytes
forever.

**Forecloses.** Hardware- or OS-derived ids (`MachineGuid`, `IOPlatformUUID`, `/etc/machine-id`), which
fail in **both** directions: they *change* on OS reinstall, so a machine becomes a stranger to its own
backups, and they *duplicate* on cloned VMs and imaged corporate laptops — two live installs silently
sharing one shard, the single failure this scheme must not have. They are also a stable cross-application
fingerprint parked in a cloud provider's folder. Also forecloses using the human label as the id, which
would leak the user's name and strand history on rename.

**Known limitation, stated not solved.** A cloned VM that copies the local config carries the same
machine id, so two live installs share a shard. Detecting this reliably needs a liveness heartbeat — a
write on every run — which is real cost for a rare event, and ADR 0010 removed the only mechanism that
would have wanted one.
