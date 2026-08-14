# Research: Gemini CLI session & state storage

Resolves [#4](https://github.com/seanrobertwright/session-backpack/issues/4).

**Evidence base:** Live inspection of `C:\Users\seanr\.gemini` (Gemini CLI v0.55.1 installed via npm), decompiled reading of the installed bundle (`@google/gemini-cli/bundle/chunk-*.js`, which inlines `packages/core/dist/src/config/storage.js`, `projectRegistry.js`, `services/keychainService.js`, `services/fileKeychain.js`, etc.), and the docs shipped inside the bundle (`bundle/docs/cli/session-management.md`, `cli/checkpointing.md`, `reference/configuration.md`). Each claim is tagged:

- **[verified-locally]** — observed on disk or reproduced (e.g., hash recomputed and matched)
- **[source]** — read directly from the installed CLI's bundled source code
- **[documented]** — from the official docs shipped with this exact version
- **[inferred]** — reasoned, not directly proven

No conversation content, tokens, or keys are reproduced here; samples are truncated/synthetic.

---

## 1. Storage locations per OS

Everything user-level lives under a single directory: **`~/.gemini`** — the path is built as `path.join(os.homedir(), ".gemini")` with no per-OS branching **[source]**. So:

| OS | User state root |
|---|---|
| Windows | `C:\Users\<user>\.gemini` |
| macOS | `/Users/<user>/.gemini` |
| Linux | `/home/<user>/.gemini` |

System-wide settings (admin/enterprise overrides, rarely present) **[documented]**:

| OS | System settings | System defaults |
|---|---|---|
| Linux | `/etc/gemini-cli/settings.json` | `/etc/gemini-cli/system-defaults.json` |
| Windows | `C:\ProgramData\gemini-cli\settings.json` | `C:\ProgramData\gemini-cli\system-defaults.json` |
| macOS | `/Library/Application Support/GeminiCli/settings.json` | `...\system-defaults.json` |

Overridable via `GEMINI_CLI_SYSTEM_SETTINGS_PATH` / `GEMINI_CLI_SYSTEM_DEFAULTS_PATH`. Project-level settings live in `<project>/.gemini/settings.json` (travels with the repo, not with `~/.gemini`).

### Top-level map of `~/.gemini` [verified-locally]

```
~/.gemini/
├── settings.json            # user settings (auth type, mcpServers, hooks, ui, retention...)
├── GEMINI.md                # global memory / context ("## Gemini Added Memories")
├── projects.json            # NEW project registry: normalized path -> slug
├── trustedFolders.json      # folder-trust decisions, keyed by absolute path
├── google_accounts.json     # { "active": "<email>", "old": [] }
├── oauth_creds.json         # OAuth token cache (plaintext JSON) — ABSENT if logged out / keychain used
├── installation_id          # random UUID identifying this install (telemetry)
├── state.json               # UI nag/tip counters
├── tmp/                     # PER-PROJECT session data (chats, logs, tool outputs, checkpoints)
│   ├── <sha256-hash>/       #   legacy dirs (pre-registry versions)
│   └── <slug>/              #   current dirs (e.g. "archon", "kanban", "archon-1")
├── history/                 # PER-PROJECT shadow git repos (file checkpointing) + shell history
│   └── <slug or hash>/
├── commands/                # user-defined slash commands
├── extensions/  agents/  skills/  hooks/  policies/ ...
└── config/                  # ancillary (not core session state on this machine)
```

Caveat observed locally: `~/.gemini` is a shared dumping ground — Google Antigravity (`antigravity*/`), third-party frameworks (`get-shit-done`, kanban backups) also write here. A Backpack adapter must use precise globs, not "grab the whole dir" **[verified-locally]**.

---

## 2. Project identity & path hashing (the critical question)

Gemini CLI has had **two generations** of project-directory naming, and this machine has both side by side **[verified-locally]**.

### Generation 1 (legacy): SHA-256 of the absolute path

`getProjectHash(projectRoot) = sha256(projectRoot)` hex — the raw path string, byte-for-byte **[source]**:

```js
function getProjectHash(projectRoot) {
  return crypto.createHash("sha256").update(projectRoot).digest("hex");
}
```

Storage was `~/.gemini/tmp/<hash>/` and `~/.gemini/history/<hash>/`.

**Verified by reproduction**: `sha256("E:\codebase\aegis")` = `cddb37643b73c2ac…`, exactly matching an on-disk `tmp/cddb3764…` dir whose sessions reference that project **[verified-locally]**. The hash is **case-sensitive and separator-sensitive** — `e:\codebase\aegis`, `E:/codebase/aegis`, and `E:\Codebase\aegis` all produce different hashes (tested).

**Answer to the critical question, for legacy dirs: YES.** A different absolute path on machine B — different drive letter, different username in the home path, different casing, or `/home/bob/proj` vs `C:\Users\alice\proj` — produces a different hash and **silently orphans the session directory**. There is no reverse index for legacy dirs; the path is unrecoverable from the hash alone (the session JSON inside does record `projectHash` but not the path).

### Generation 2 (current, v0.55.x): slug registry in `projects.json`

Current versions use a **ProjectRegistry** (`~/.gemini/projects.json`) mapping *normalized* project path → human-readable slug **[source, verified-locally]**:

```json
{ "projects": {
    "e:\\projects\\archon": "archon",
    "e:\\dynamous\\archon": "archon-1",
    "c:\\users\\seanr\\onedrive\\documents\\scrap\\kanban": "kanban"
} }
```

- **Normalization**: `path.resolve(p)`, then `.toLowerCase()` **on Windows only** (`os.platform() === "win32"`). macOS and Linux keep the case as-is in the registry **[source]**.
- **Slug**: slugified `basename(projectPath)` (lowercase, `[a-z0-9-]`), deduplicated with `-1`, `-2`… suffixes across all projects (`e:\projects\archon` → `archon`, `e:\dynamous\archon` → `archon-1`) **[source, verified-locally]**.
- **Ownership marker**: each `tmp/<slug>/` and `history/<slug>/` contains a `.project_root` file holding the normalized owning path (e.g. `e:\projects\archon`). On startup, `getShortId()` verifies the slug still "belongs" to the path; if the marker disagrees, the registry entry is **deleted and a fresh slug is claimed** **[source]**.
- **Migration**: on first run of a new version, legacy `tmp/<hash>` content is copied into `tmp/<slug>` **[source]** — which is why this machine has both forms.
- Registry writes are guarded with `proper-lockfile` locking **[source]**.

**Answer for current versions: still YES, but recoverable.** A different absolute path on machine B simply isn't in `projects.json`, so the CLI mints a *new* (possibly identical-basename `foo` → but if the copied dir's `.project_root` disagrees, collision detection forces `foo-1`) slug and starts empty — the restored sessions sit orphaned under the old slug. **However**, because identity is now an editable registry + marker files rather than a one-way hash, an adapter can re-home a project (see §7).

Note: session headers still embed a `projectHash` field (sha256 of the runtime project root) **[source]**, but since the registry migration it is informational; the *directory slug* is the lookup key. On this machine, sessions recorded via an external driver (Archon's a2a-server, custom `--session-id`) carry hashes that don't match the marker path — consistent with a different cwd (git worktree) at runtime **[inferred]**.

---

## 3. Session files: formats & schemas

Per project: `~/.gemini/tmp/<id>/chats/` **[verified-locally, documented]**.

**Filename**: `session-<YYYY-MM-DDTHH-MM>-<first 8 chars of sessionId>.json` (legacy) or `.jsonl` (current). Session IDs are UUIDv4 by default, or any string supplied via `--session-id` (observed: `a2a-server` → filename fragment `a2a-serv`) **[verified-locally]**.

**Legacy `.json`** — a single document, rewritten on update:

```json
{
  "sessionId": "38747c34-…",
  "projectHash": "cddb3764…",
  "startTime": "2026-02-14T02:08:40.030Z",
  "lastUpdated": "…",
  "messages": [
    { "id": "<uuid>", "timestamp": "…", "type": "user",
      "content": [ { "text": "<prompt text>" } ] }
  ]
}
```

**Current `.jsonl`** — append-only log; line 1 is a header record, then message records, plus MongoDB-style patch records:

```jsonl
{"sessionId":"76fd2e94-…","projectHash":"3e289428…","startTime":"…","lastUpdated":"…","kind":"main"}
{"id":"<uuid>","timestamp":"…","type":"info","content":"…"}
{"$set":{"lastUpdated":"…"}}
```

Recorded content: full conversation (prompts, responses), all tool calls with inputs/outputs, token-usage stats, and model thoughts/reasoning summaries **[documented]**. Files grow large — a 10.9 MB single-session file was observed **[verified-locally]**. Sessions store **plaintext conversation and tool output** — treat as sensitive.

Sibling per-project artifacts **[verified-locally, source]**:

| Path (under `tmp/<id>/`) | Contents |
|---|---|
| `chats/session-*.json(l)` | conversation transcripts (the resumable state) |
| `checkpoint-<tag>.json` | manual chat checkpoints from `/chat save <tag>` |
| `logs.json` (legacy) / `logs/session-<id>.jsonl` | user-prompt activity log |
| `tool-outputs/` (or legacy `tool_output/`) | spill files for truncated tool outputs, `session-<id>/` subdirs |
| `shell_history` | per-project shell command history for the `!` shell |
| `checkpoints/` | file-restore checkpoint metadata (when checkpointing enabled) |
| `memory/`, `tasks/`, `plans/` | project memory / task-tracker state (newer features) |
| `.project_root` | ownership marker (normalized project path) |

**Retention warning:** by default Gemini CLI **auto-deletes sessions older than 30 days** (`general.sessionRetention`, `enabled: true`, `maxAge: "30d"`, optional `maxCount`; `minRetention` floor 1d). Deletion cascades to plans, task trackers, tool outputs, and logs **[documented]**. This machine raises it to `120d` **[verified-locally]**. A backup tool cannot assume history persists.

---

## 4. Resume mechanics

**Automatic recording** — every interactive session is recorded to `chats/` with no user action **[documented, verified-locally]**.

**CLI flags** (v0.55.1 `--help`, **[verified-locally]**):

- `gemini --resume` / `--resume latest` — resume most recent session for *this project* (dir chosen by cwd → registry lookup)
- `--resume <index>` — index from `--list-sessions` (sessions sorted by file mtime, newest first **[source]**)
- `--resume <uuid>` — resume by full session ID **[documented]**
- `--list-sessions`, `--delete-session <index|id>`
- `--session-id <uuid>` — start a new session under a chosen ID (external orchestrators use this)
- `--session-file <path>` — **load a session from an arbitrary JSON file** — a gift for Backpack restores: no registry surgery needed to rehydrate one conversation **[verified-locally: flag exists; behavior documented]**

**In-session** — `/resume` opens an interactive Session Browser (browse/preview/search/delete); `/chat save <tag>` writes `tmp/<id>/checkpoint-<tag>.json`, `/chat resume <tag>` restores it; `/chat list`, `/chat delete`, plus `/resume save|list|resume` aliases **[documented, source]**.

**File checkpointing** (`/restore`) — separate, opt-in (`general.checkpointing.enabled`). Before file-modifying tools run, the CLI commits a snapshot of the *project working tree* into a **shadow git repo at `~/.gemini/history/<id>/`** (observed locally: `.git`, `.gitconfig`, `.gitignore`) and stores conversation+tool-call metadata under `tmp/<id>/checkpoints/`. `/restore` reverts working-tree files and conversation together **[documented, verified-locally]**.

**Session identity in one sentence:** a session = one `session-*.json(l)` file, identified by `sessionId`, findable only through its project directory (`tmp/<slug>` via cwd → `projects.json` lookup); resume = re-reading that file back into context.

---

## 5. Memory (GEMINI.md)

- **Global**: `~/.gemini/GEMINI.md`. The `save_memory` tool appends bullets under a `## Gemini Added Memories` heading **[verified-locally]**. Fully portable plain Markdown — prime Backpack cargo.
- **Project**: `GEMINI.md` files discovered hierarchically from cwd up to the project root and downward in subdirectories; these live *in the repo*, so they travel with normal code sync, not with the vault **[documented]**.
- Filename is configurable (`context.fileName`), and extensions can contribute context files **[documented, source]**.

---

## 6. Where secrets live

| Secret | Location | Portable? |
|---|---|---|
| Google OAuth token cache (default "Login with Google") | `~/.gemini/oauth_creds.json` — **plaintext JSON** containing access + refresh token **[source]** | Technically copyable, but **never vault it** — it grants account access; machine-B login takes seconds |
| OAuth cache (opt-in `GEMINI_FORCE_ENCRYPTED_FILE_STORAGE=true`) | OS keychain via keytar — service `gemini-cli-oauth`, account `main-account` (Windows Credential Manager / macOS Keychain / libsecret) **[source]** | **No** — not on the filesystem at all |
| Keychain file fallback | `~/.gemini/gemini-credentials.json`, AES-256-GCM, key = `scrypt("gemini-cli-oauth", salt = hostname-username-gemini-cli)` **[source]** | **No — cryptographically machine-bound**: different hostname/username cannot derive the key |
| API keys (`GEMINI_API_KEY`, `GOOGLE_API_KEY`, Vertex vars) | environment / `.env` files (cwd upward, then `~/.env`) **[documented]** | Out of scope for `~/.gemini` capture |
| MCP server OAuth tokens | keychain (per-server accounts) with file fallback **[source]** | No |
| Account identity (email only, no token) | `~/.gemini/google_accounts.json` **[verified-locally]** | Yes (low sensitivity) |

On this machine `oauth_creds.json` is absent and running `gemini --list-sessions` immediately prompted for browser re-auth **[verified-locally]** — confirming: no creds file ⇒ re-login, and nothing else in the vault breaks.

Also note: chat transcripts themselves frequently contain secrets echoed through tool output (env dumps, config files). The vault's encryption is doing real work here.

---

## 7. Portable vs machine-specific, and what must travel

### Must travel (the resumable session state)

| Item | Path |
|---|---|
| Conversation transcripts | `tmp/<id>/chats/session-*.json(l)` |
| Manual chat checkpoints | `tmp/<id>/checkpoint-*.json` |
| Project shell history | `tmp/<id>/shell_history` |
| Project memory/plans/tasks | `tmp/<id>/memory/`, `plans/`, `tasks/` (if present) |
| Ownership marker | `tmp/<id>/.project_root` (must be **rewritten** on restore) |
| Registry entry | the project's key/value in `projects.json` (merge, don't overwrite) |
| Global memory | `~/.gemini/GEMINI.md` |
| Nice-to-have config | `settings.json` (see caveats), `commands/`, custom `agents/`/`skills/` |

### Should NOT travel

- `oauth_creds.json`, `gemini-credentials.json` — secrets; the latter won't decrypt anyway
- `installation_id` — machine identity for telemetry; cloning it conflates machines
- `trustedFolders.json` — keyed by machine-local absolute paths; auto-trusting paths on a new machine is a (mild) security decision the user should re-make
- `history/<id>/` shadow git repos — snapshots of the *old machine's* working tree; restoring them against a different checkout invites bad `/restore` results. Skip (or archive-only, never restore).
- `tmp/<id>/tool-outputs/`, `logs/` — bulky, derivative; optional archive-only
- `extensions/`, `antigravity*`, `bin/` — installed artifacts; reinstall instead
- `state.json`, `*.bak` — trivia

### Restore recipe for machine B (project at a *different* path)

1. Copy `tmp/<slug>/` (chats, checkpoints, shell_history, memory) into `~/.gemini/tmp/<slug>/`.
2. Compute machine B's normalized path: `path.resolve(newProjectRoot)`, lowercased **if Windows**.
3. Upsert `projects.json`: `{ "<normalizedNewPath>": "<slug>" }` (file-lock friendly: write-temp-rename; if the slug is taken by another path on B, pick a free one and rename the dir to match).
4. Overwrite `tmp/<slug>/.project_root` with the normalized new path. (If skipped, `verifySlugOwnership` drops the mapping and the CLI mints a fresh empty slug — the restore silently fails **[source]**.)
5. Do the same for `history/<slug>/.project_root` only if you chose to carry the shadow repo (not recommended).
6. `cd <newProjectRoot> && gemini --list-sessions` → `gemini --resume latest`.

If the project sits at the **identical absolute path** (and identical casing on Linux) on machine B, steps 2–5 collapse to "copy the dir and merge the registry line". For legacy hash-named dirs the only options are: restore the project to the exact same path, or rename the dir to `sha256(newPath)`.

Escape hatch: for one-off restores, skip registry surgery entirely and hand the transcript to `gemini --session-file <copied-session.json>` **[verified-locally: flag exists]**.

---

## 8. Implications for a Backpack adapter

**Capture globs** (relative to `~/.gemini`):

```
tmp/*/chats/*.json
tmp/*/chats/*.jsonl
tmp/*/checkpoint-*.json
tmp/*/.project_root
tmp/*/shell_history
tmp/*/memory/**
projects.json
GEMINI.md
settings.json          # optional, sanitize first
commands/**            # optional
```

Exclude: `tmp/*/tool-outputs/**`, `tmp/*/logs/**`, `tmp/bin/**`, `history/**`, `antigravity*/**`, `extensions/**`, `oauth_creds.json`, `gemini-credentials.json`, `installation_id`, `trustedFolders.json`, `*.bak`.

**Watch paths**: `~/.gemini/tmp/*/chats/` is the high-signal watch target — `.jsonl` files are append-mostly, updated on every turn; a session's `mtime` is its "last activity". `projects.json` changes signal new projects. Debounce: single-session `.json` files are fully rewritten and can be multi-MB.

**Restore steps**: as §7. The adapter needs a tiny amount of logic (normalize path per-OS, upsert registry JSON, rewrite two marker files) — no binary formats, no databases, all JSON/JSONL/Markdown. This makes Gemini CLI one of the *easier* adapters.

**Risks / gotchas**:

1. **Retention race** — default 30-day auto-delete (cascade) means Backpack must capture promptly and must not "restore" a session onto a machine where the CLI then immediately garbage-collects it (check target's `sessionRetention` vs session age).
2. **Version skew** — the hash→slug migration happened recently; a machine-B CLI older than the registry feature won't read slug dirs. Pin/verify CLI versions across machines. Format is not a stable public API.
3. **`.project_root`/registry consistency** — a restore that copies dirs but not markers fails *silently* (fresh empty slug). Adapter must treat registry+markers+dirs as one atomic unit.
4. **Windows-only lowercasing** — path normalization differs per OS; adapter must reimplement it exactly (resolve → lowercase iff win32).
5. **Slug collisions** — machine B may already own the slug (`archon` taken → use `archon-1` and rename dir) — adapter needs the same dedupe loop the CLI uses.
6. **Secrets in transcripts** — transcripts embed tool output (env vars, file contents). Vault-side encryption is mandatory; also consider marking the Gemini adapter's payloads as sensitive by default.
7. **Shared `~/.gemini`** — other tools (Antigravity IDE, third-party frameworks) colonize this directory; strict allowlists, never `tmp/**` wholesale (observed non-CLI junk dirs inside `tmp/`).
8. **Concurrent writes** — the CLI locks `projects.json` (proper-lockfile); Backpack restores should honor the same lockfile convention or restore only while the CLI isn't running.
9. **Auth is out of scope** — after restore the user just logs in again on machine B (browser OAuth or `GEMINI_API_KEY`); do not attempt to move credentials.

---

## Appendix: version note

All source-level claims verified against **Gemini CLI v0.55.1** (npm global install, Windows). The legacy hash scheme was observed in directories created by earlier versions (2025-09 → 2026-02); the slug registry appears in directories from 2026-03 onward on this machine.
