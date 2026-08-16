# 0006. Backpack never rewrites file contents

- **Status:** accepted
- **Date:** 2026-08-16
- **Settled by:** [#12](https://github.com/seanrobertwright/session-backpack/issues/12), [#13](https://github.com/seanrobertwright/session-backpack/issues/13), [#21](https://github.com/seanrobertwright/session-backpack/issues/21) · **Spec:** [v1 §4.2, §5.6, §7.2](../spec/v1.md)

## Context

The same temptation arrives from three unrelated directions, and each time it looks locally reasonable:

- **Capture** — a mid-write capture may end in a torn record. Trimming to the last valid line would
  produce a clean file.
- **Restore** — a restored session's id may collide, or its identifier may not suit the target machine.
  Minting a new id would make every restore purely additive.
- **Toolstate restore** — a restored MCP config points at `C:\Users\sean\…` and will not work on a Mac.
  Rewriting the path would make it work.

Each is a small, well-intentioned edit. Together they would make Backpack a tool that changes what it
gives you back.

## Decision

> **Backpack rewrites *placement*, never *contents*.**

- **Capture copies bytes.** Never truncate to the last valid record; a torn tail is stored as-is and
  surfaces loudly in the reader (`Body::Raw`), and the next capture supersedes it.
- **Restore re-homes.** It rewrites which file lands where and which registry points at it. It
  **preserves the tool's native session id** wherever the identifier survives re-homing, re-keying only
  where the id is path-derived and preservation is physically unavailable (Gemini). Provenance is
  stamped as `restored_from`.
- **Toolstate restores verbatim.** Backpack **warns** about machine-specific absolute paths; it does not
  fix them.

## Consequences

**Makes easy.** Byte-exact restore, which is the whole value proposition. Capture needs no atomicity
mechanism at all, because a record-boundary prefix is a *correct capture of an earlier moment* rather
than damage. A user can always diff what came back against what went in.

**Makes hard.** Claude Code carries `sessionId` on every JSONL line, so minting a new id would mean
rewriting contents — which makes `KeepBoth` at restore an explicit, user-chosen exception rather than a
default. Restored configs may not work on the target machine until the user edits them.

**Forecloses.** Making restored configs portable (out of scope, spec §12) — a real feature, but a
different one. Also any "repair" of a damaged transcript.

**Why it is hard to reverse.** The value of a backup tool is that what comes back is what went in. Once
Backpack is known to edit contents in *any* case, every restore becomes something the user must verify,
and the guarantee cannot be re-established by a later release.
