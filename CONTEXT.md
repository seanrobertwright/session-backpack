# CONTEXT — Backpack's ubiquitous language

Every domain term Backpack uses, defined once. **Read this before writing anything that names a
domain concept, and use these terms verbatim.** If a term you need is missing or fuzzy, sharpen it
here via `domain-modeling` rather than inventing a synonym.

The design these terms describe is specified in [`docs/spec/v1.md`](docs/spec/v1.md) and was
settled on [Wayfinder map: Backpack v1 spec #1](https://github.com/seanrobertwright/session-backpack/issues/1).

---

## The product

**Backpack**
: A desktop app that backs up, browses, restores and exports AI agent sessions into an encrypted
  **vault** living in a folder the user already syncs. Positioned as *Time Machine for AI agent
  sessions*.

**BYO sync**
: Bring-your-own sync. Backpack operates no server and holds no account. The vault is an ordinary
  directory the user places inside OneDrive, Dropbox, Google Drive, Syncthing, or a git repo, and
  their existing sync tool moves the bytes. Backpack never talks to a sync provider.

---

## Storage

**Vault**
: The complete archive: one directory, fully self-describing, safe to place in a synced folder.
  Append-only — nothing in it is ever rewritten. See spec §3.

**Blob**
: One captured file, zstd-compressed and age-encrypted, stored immutably at
  `blobs/<hh>/<hash>.age` where the name is the **BLAKE3 hash of the plaintext**. The name *is* the
  checksum, so integrity verification is free. Blobs are the only files Backpack ever deletes, and
  only under a **proof chain**.

**Manifest**
: A small encrypted JSON document describing one **capture** — which blobs compose it, listing
  metadata, machine, timestamps. Write-once and **never deleted**. A blob without a manifest is
  meaningless, which is why manifests are the precious half of the vault and blobs the cheap half.

**Native bytes**
: Exactly what the agent tool wrote, byte for byte. Blobs hold native bytes only; no rendering,
  normalization or projection is ever persisted in the vault.

**Recipient**
: An age public key a blob or manifest is encrypted to. Every file is encrypted to two: the vault's
  primary **identity** and the **recovery kit**.

**Identity**
: The vault's X25519 private key. Stored in the vault at `identity.age`, wrapped under the user's
  passphrase; cached decrypted in the OS keychain per machine when *Remember on this device* is on.

**Recovery kit**
: A second age identity generated at vault creation, shown once, never stored by Backpack. Every
  file is encrypted to it from byte one, so it can open the vault without the passphrase.

---

## Time and identity

**Session**
: The agent tool's own conversation, as the tool understands it — one Claude Code transcript UUID,
  one Codex rollout, one Gemini chat. **One session, however many machines touch it.** Identified
  in the vault by an **opaque id**.

**Opaque id**
: `blake3(id_salt || tool || native_id)`, 128-bit hex — the session's directory name in the vault.
  Hides the tool's native identifier (and therefore any project path baked into it) from anyone who
  can list the synced folder.

**Native id**
: The tool's own session identifier — the UUID in the filename, the `sessionId` in the header.
  Always a session-level, high-entropy value; **never** a path-derived hash.

**Capture**
: One point-in-time version of a session: one manifest plus the blobs it names. Captures are
  permanent and additive. "Backing up" mints a capture.

**Branch**
: One machine's line of captures of a session — `sessions/<tool>/<opaque-id>/<machine>/`. Structural
  in the layout, not an inference. A session usually has exactly one branch.

**Fork**
: Two branches of one session that both advanced past their shared ancestor. Backpack **shows** forks
  and lets the user choose between them; it never merges them.

**Fold**
: The opposite of a fork — two branches recognised as one story (identical tip blob hashes, or one
  descended from the other via `restored_from`) and shown as a single session entry. Classified from
  manifest evidence alone; no blob is read.

**Machine**
: **One install of Backpack**, identified by a random 128-bit opaque id minted at first run and held
  in machine-local config. Never hardware-derived. Reinstalling mints a new machine.

**Shard**
: The part of the vault a given machine writes — its own subdirectory under every session, plus
  `machines/<id>/`. Per-machine sharding is what makes concurrent writes physically unable to
  collide.

**Toolstate**
: A tool's non-session state — settings, MCP configs, memory files. Belongs to the *tool*, not to
  any session, and lives under `state/<tool>/`. Restored per **file role**, never merged.

---

## Adapters

**Adapter**
: A near-pure description of one agent tool: where its sessions live, which files compose one, and
  how to parse those files into **turns**. An adapter never touches the vault — the core owns every
  read, hash, compression, encryption and write. **Restore is the single carved-out exception.**

**Core**
: Everything that is not an adapter or UI: the watcher, capture pipeline, vault, crypto, index and
  compactor. The core is trusted; adapters are not.

**Install**
: One installation of one tool on this machine. An adapter's `detect()` returns zero or more —
  VS Code Stable and Insiders are two installs of one adapter, not two adapters.

**Capture plan**
: An adapter's **allow-list**: a root plus the exact files and globs composing one session or one
  tool's state. Nothing is captured implicitly.

**Deny list**
: A core-owned list of paths that may never be captured — credential files, keys, `.env`. **No
  adapter can override it**, and a plan that hits it fails loudly rather than silently dropping the
  file.

**File role** (`FileRole`)
: What a captured file *is*, and therefore how restore and the reader treat it: `Transcript`,
  `Subtranscript`, `Sidecar`, `Settings`, `Mcp`, `Memory`, `Unknown`.

**Sidecar**
: A file restored verbatim and never projected — todos, checkpoints, artifact directories.

**Subtranscript**
: A subagent's own transcript: projected and readable, but never concatenated into the main stream.

**Re-homing**
: Restore rewriting a session's **placement** — which directory it lands in, which registry points
  at it — so the target tool finds it on this machine. Restore re-homes; it never rewrites file
  *contents*.

**Restorable**
: Whether an adapter can restore. **Computed** from `restorer().is_some()` and stamped onto every
  manifest at capture time, so a session captured while an adapter was capture-only stays honestly
  labelled after that adapter graduates.

**Capture-only**
: An adapter that captures but cannot restore. A statement about **missing evidence**, not a
  permanent tier.

---

## Reading

**Projection**
: The normalized, tool-agnostic rendering of a transcript: a sequence of **turns** plus an optional
  label. Derived **at read time** from native bytes and cached per machine; never stored in the
  vault. A parser fix therefore retroactively improves every session ever captured.

**Turn**
: One record in a projection — a role, an optional timestamp, and a **body**.

**Turn role** (`TurnRole`)
: Who spoke: `user`, `assistant`, `system`, `tool`. Distinct vocabulary from **file role**.

**Faithful superset**
: The projection's governing rule — `project()` emits a turn for *every* record in the file and the
  **reader** filters. What an adapter drops is gone until a new adapter version ships; what the
  reader hides is a checkbox.

**Opaque** (`Body::Opaque`)
: A record deliberately not decoded because it is known machinery. Hidden by default, revealable,
  carries its native type name.

**Raw** (`Body::Raw`)
: A record the parser **failed** to decode. **Always visible.** `Opaque` and `Raw` are separate
  variants precisely so a parser bug can never disguise itself as noise.

**Partial**
: A quiet advisory badge on a session whose capture-time projection contained raw or opaque turns.
  Pessimistic-only — it can over-warn, never under-warn — and self-corrects when the session is
  opened.

**Sanitized export**
: A plaintext, shareable rendering of transcripts only, with known secret shapes redacted, shown in
  a default-on review pane before anything is written. **Fails closed**: images, opaque records and
  unparsed bytes are omitted behind a counted stub rather than exported unredacted.

---

## Capture cadence

**Settle**
: The 90-second quiet period after writes to a session stop, after which a capture fires. The only
  trigger in normal operation.

**Floor**
: The 10-minute minimum between two captures of the same session.

**Stability check**
: A stat → read → stat bracket over *every file of one session*. Any movement, or any read failure,
  **discards the whole attempt and re-arms the settle timer** — never a retry loop, never a partial
  capture written.

**Record-boundary prefix**
: A capture containing the first N complete records of a growing file. **Not corruption — a correct
  capture of an earlier moment.** This is why capture needs no atomicity mechanism.

---

## Compaction

**Compaction**
: Deleting blobs that carry no unique information, because their bytes survive inside a blob that
  still exists. **Lossless** — every capture stays byte-exact readable. Runs weekly, invisibly, on
  a machine's own shard only.

**Retention**
: Deliberately forgetting history the user might still want. **Backpack has none.** The word is
  reserved so nobody uses it for compaction.

**Prefix proof** (`supersedes`)
: A per-file manifest field, `{ blob, len }`, recorded **forward onto the successor** at capture
  time by the core, stating that the predecessor blob's bytes are the first `len` bytes of this one.
  An observation made against real bytes, never an adapter's claim.

**Proof chain**
: The transitive sequence of prefix proofs that reconstructs a deleted blob from a surviving one.
  The governing invariant: *a blob may be deleted iff a surviving proof chain reconstructs it from a
  blob that still exists.*

**Absent by design**
: A verify-on-read state distinct from *missing*: the blob is gone, a proof chain explains why, and
  the reader reconstructs it. Never reported as corruption.

---

## Interface

**View shape**
: The user's global choice of navigation shape — **Console**, **Navigator** or **Timeline** —
  governing both Home and Sessions. One setting for the whole app.

**Shell**
: Everything outside the view shape: the rail, the filter facets, the current filter, the selected
  session. Switching shape never changes place.

**Skin**
: A theme — **Auto** (default, follows the OS), **Slate**, **Paper**, **Terminal**. Mechanically a
  re-declaration of CSS custom properties behind `[data-skin]`.

**Semantic colour**
: Backed-up / attention / failed. A **separate axis** from the accent colour, so status never
  competes with branding.

---

## Words we deliberately do not use

- **Garbage collection / GC** — Backpack has a **compactor**, and it never asks who references a
  blob. Using "GC" imports mark-and-sweep assumptions that do not apply.
- **Sync** (of Backpack's own doing) — Backpack writes files; the *user's* sync tool syncs them.
- **Delete** (of history) — nothing is ever forgotten. Blobs are **compacted**; captures are
  **superseded**.
- **Merge** (of sessions) — forks are **shown**, never merged.
