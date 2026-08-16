# 0005. Adapters are untrusted; the core owns every byte

- **Status:** accepted
- **Date:** 2026-08-16
- **Settled by:** [#12](https://github.com/seanrobertwright/session-backpack/issues/12) · **Spec:** [v1 §5.1, §5.2](../spec/v1.md)

## Context

Nine adapters at v1 and more later, each describing a tool whose format churns independently. Adapters
are the part of the system most likely to be wrong, most likely to change, and — eventually — most likely
to be written by someone other than us. They also operate on directories that contain **plaintext OAuth
refresh tokens and API keys**, and they write into a vault that sits in the user's cloud-synced folder.

An adapter bug must not be able to leak a credential into a cloud provider's storage, permanently.

## Decision

**An adapter is a near-pure description of one tool, never an agent that acts on the vault.**

- `plan_capture()` and `project()` are **pure functions**: no I/O, structurally incapable of writing
  plaintext anywhere. The core resolves paths, reads, hashes, compresses, encrypts and writes.
- **Capture is an explicit allow-list.** Nothing is captured implicitly.
- Every resolved path is vetted against a **core deny list no adapter can override**. A plan that hits it
  **fails hard**, rather than silently dropping the file.
- **Restore is the single carved-out exception**, because the evidence says so: every tool's capture is
  *"copy these files"*, but every tool's restore is different work. Even then, the core decrypts and
  verifies into a staging area first, so `Restorer` never touches the vault.

## Consequences

**Makes easy.** Adapters are trivially unit-testable against a fixture directory. Age, zstd, hashing and
secret handling exist once, not nine times. A new adapter is one module of pure functions plus one
registry line, needing zero changes to capture, vault, encryption, watching or UI.

**Makes hard.** Adapters cannot express capture logic requiring I/O — a genuine constraint, accepted
because `detect` and `enumerate` remain impure for exactly the cases that need it. Allow-list capture
**rots**: a tool that starts writing a newly-useful file is missed until the next release.

That asymmetry is the point. Under ADR 0001 the vault is append-only, so **a missed file is a
next-release bug while a captured secret is unfixable forever.** A deny-list capture root would rot more
slowly and fail permanently.

**Forecloses.** Dynamically-loaded third-party adapter plugins, which fight this boundary's premise
directly. Also any design where an adapter's *declaration* gates a destructive operation — which is why
compaction's prefix proof is an observation made by the core against real bytes, not an adapter's claim
that a role is append-only (ADR 0010).
