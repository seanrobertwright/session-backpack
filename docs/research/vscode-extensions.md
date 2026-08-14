# Research: VS Code AI Extension Session Storage

**Ticket:** [#6](https://github.com/seanrobertwright/session-backpack/issues/6)
**Date:** 2026-08-14
**Method:** Primary evidence is direct inspection of the live VS Code profile at `C:\Users\seanr\AppData\Roaming\Code` (VS Code 1.128.x, Copilot Chat 0.56.x era) plus `~/.claude`. Copilot Chat and the Claude Code extension were inspected locally; **Cline and Roo Code are not installed on this machine**, so those sections are from docs/issues/community sources. Every claim is tagged: `[verified-locally]`, `[documented]`, or `[inferred]`.

All sample values below are redacted or synthetic. No conversation content, tokens, or keys appear in this document.

---

## 1. VS Code storage roots (per OS)

| OS / context | User data root |
|---|---|
| Windows | `%APPDATA%\Code\User` (`C:\Users\<user>\AppData\Roaming\Code\User`) `[verified-locally]` |
| macOS | `~/Library/Application Support/Code/User` `[documented]` |
| Linux | `~/.config/Code/User` `[documented]` |
| Remote-SSH / WSL / devcontainer | `~/.vscode-server/data/User` **on the remote host** — invisible to a Windows-side scanner unless it reaches into `\\wsl$\...` `[documented]` |

Product variants swap the `Code` segment: `Code - Insiders`, `VSCodium`, `Cursor`, `Windsurf` all use the same internal layout. `[documented]`

Under the root, the two trees that matter:

- `User/workspaceStorage/<hash>/` — per-workspace state. Contains `workspace.json`, `state.vscdb` (+ `.backup`), `chatSessions/`, `chatEditingSessions/`, and one folder per extension that used workspace-scoped storage (e.g. `GitHub.copilot-chat/`). `[verified-locally]`
- `User/globalStorage/` — global `state.vscdb` (+ `.backup`), `storage.json`, `emptyWindowChatSessions/`, and one folder per extension (`github.copilot-chat/`, `saoudrizwan.claude-dev/`, `rooveterinaryinc.roo-cline/`, ...). `[verified-locally]` (155 workspace hash dirs present on this machine.)

### state.vscdb format `[verified-locally]`

Plain SQLite 3, single table:

```sql
CREATE TABLE ItemTable (key TEXT UNIQUE ON CONFLICT REPLACE, value BLOB)
```

Values are JSON strings (or JSON-wrapped byte buffers for secrets, §6). Both the workspace-level and global-level `state.vscdb` use this identical schema. A sibling `state.vscdb.backup` is maintained by VS Code.

**Critical operational fact:** VS Code's storage service keeps `ItemTable` cached in memory while running and flushes on idle/shutdown. External writes made while VS Code is open can be silently clobbered on exit, in addition to plain SQLite locking hazards. `[documented]` (VS Code storage service behavior; locking observed as WAL/journal siblings locally.)

---

## 2. workspaceStorage hash mechanics — the orphaning problem

The `<hash>` directory name for a local single-folder workspace is:

```
md5( folderFsPath + String(folderCreationTimeMs) )
```

`[verified-locally]` — reproduced exactly on two real workspaces on this machine:

- `md5("c:\Users\...\gemma4" + "1780183393573")` → `0620417cd682c717f2398a375a5612bd` (matches dir name; `1780183393573` is the folder's birthtime in ms)
- `md5("e:\codebase\gsd" + "1770762560732")` → `212cf86d7b19803e104b7ff849bdca1d`

Notes on the exact recipe `[verified-locally]`:
- The path is the URI `fsPath` form: backslashes, **lowercase drive letter**, no trailing slash.
- The timestamp is the folder **creation time** (birthtime) in integer milliseconds, string-concatenated after the path.

Per VS Code source (`src/vs/platform/workspaces/node/workspaces.ts`) the time component is: Windows/macOS → birthtime, **Linux → inode number**; remote (non-`file:`) URIs hash the full URI string instead. Multi-root `.code-workspace` files hash the config file path the same way. `[documented]`

Each hash dir contains a `workspace.json` reverse map, e.g.:

```json
{ "folder": "file:///c%3A/Users/<user>/OneDrive/Documents/<project>" }
```

`[verified-locally]` — this file is for reverse lookup only; VS Code recomputes the hash at open time, it does not consult `workspace.json` to find the dir.

**Consequence for Backpack:** the hash can **never** match across machines — even with a byte-identical path on machine B, the folder's creation time (or inode on Linux) differs. Every workspace-scoped session (all Copilot Chat history) is orphaned by a naive file copy. A restore must re-key the data:

1. Ensure the project folder exists on machine B.
2. Either (a) compute machine B's hash = `md5(lowercased-drive fsPath + birthtimeMs)`, or (b) more robustly, have the user open the folder once in VS Code (which creates the dir), then locate it by scanning `workspaceStorage/*/workspace.json` for the matching folder URI.
3. Copy session payloads into that dir and merge the index keys into its `state.vscdb` (§3).

A different path on machine B is fine **as long as the re-keying targets whatever hash machine B actually produces** — the session JSONL files themselves do not embed the workspace path. `[verified-locally]` (inspected session files; no workspace path fields in the session snapshot itself)

---

## 3. GitHub Copilot Chat

Two owners of state, which surprises people: **VS Code core owns the chat transcripts**; the Copilot extension owns only auxiliary data.

### 3a. Core chat sessions (the actual conversations) `[verified-locally]`

- Per workspace: `workspaceStorage/<hash>/chatSessions/<sessionUuid>.jsonl`
- Chats opened in an empty window: `globalStorage/emptyWindowChatSessions/<sessionUuid>.jsonl`
- Filename UUID == `sessionId` field inside == the session's identity.

Format: a **log-structured JSONL** (schema `version: 3`). Line kinds observed:

- `{"kind":0,"v":{...}}` — full session snapshot: `version`, `creationDate`, `initialLocation` ("panel"), `responderUsername`, `sessionId`, `hasPendingEdits`, `requests[]`, `inputState` (attachments, mode `{"id":"agent","kind":"agent"}`, `selectedModel` incl. model metadata).
- `{"kind":1,"k":[<json-path>],"v":...}` — set value at path, e.g. `k:["customTitle"]`, `k:["requests",0,"result"]`, `k:["requests",0,"modelState"]`.
- `{"kind":2,"k":[<json-path>],"v":[...]}` — append at path, e.g. `k:["requests"]` (new request with `requestId: "request_<uuid>"`, `timestamp`, message) and `k:["requests",0,"response"]` (response parts incl. `{"kind":"thinking",...}`, markdown parts, tool calls).

Replaying the log reconstructs the session; a real 153-line file (~6.8 MB) was inspected. Fully self-contained text — **this is the portable payload.**

- Edit checkpoints: `workspaceStorage/<hash>/chatEditingSessions/<sessionUuid>/state.json` + `contents/` (file snapshots). `state.json` `version: 2`: `initialFileContents`, `timeline.checkpoints[]` (`checkpointId`, `epoch`, `label`), `fileBaselines`, `operations`, `recentSnapshot`. `[verified-locally]`
- Session index: `state.vscdb` key **`chat.ChatSessionStore.index`** (workspace db for workspace chats, global db for empty-window chats): `{"version":1,"entries":{"<sessionUuid>":{"sessionId","title","lastMessageDate","timing","initialLocation","hasPendingEdits","isEmpty","isExternal","lastResponseState","permissionLevel"}}}`. `[verified-locally]` The history picker is driven by this index, so restored JSONL files without matching index entries are unlikely to be listed. `[inferred]`
- Legacy note: before the JSONL store (≈2025), sessions lived entirely inside `state.vscdb` (`interactive.sessions` key). Scanned all 155 workspace DBs on this machine: **zero** legacy keys remain — everything migrated. Modern Backpack can target JSONL and treat the DB as index-only. `[verified-locally]`

### 3b. Copilot extension's own storage `[verified-locally]`

- `workspaceStorage/<hash>/GitHub.copilot-chat/`:
  - `transcripts/<sessionUuid>.jsonl` — flat event log (`session.start` with producer/version fields, `user.message`, ... each with `id`/`timestamp`/`parentId`). Duplicates conversation content in simpler form; useful secondary capture.
  - `chat-session-resources/<sessionUuid>/` — attachments/resources per session.
  - `codebase-external.sqlite` — local embeddings/index cache, regenerable, **exclude** from backup.
  - `debug-logs/<sessionUuid>/models.json`.
- `globalStorage/github.copilot-chat/` — caches only: `commandEmbeddings.json`, `settingEmbeddings.json` (~40 MB combined), `toolEmbeddingsCache.bin`, CLI shims (`copilotCli/`), agent prompt files, `vscode-sessions-<uuid>/diff.index`. Nothing conversational worth backing up.
- Extension mementos: workspace `state.vscdb` key `GitHub.copilot-chat` (nonce, per-workspace CLI session file pointer); global keys `GitHub.copilot`, `github.copilot-chat-github-*` (account metadata, not the token).

### 3c. Auth
Copilot signs in through VS Code's GitHub authentication provider; the token lives in VS Code secret storage (§6). Machine-bound; never portable; user re-authenticates on machine B. `[verified-locally]` (secret keys present in global db) / `[documented]` (flow)

---

## 4. Cline and Roo Code — `[documented]` (not installed on this machine; confirmed absent from `globalStorage/`)

Both store **tasks globally, not per workspace** — so they dodge the workspace-hash orphaning problem entirely.

### Cline (`saoudrizwan.claude-dev`)

- `globalStorage/saoudrizwan.claude-dev/`
  - `tasks/<taskId>/` — taskId is an epoch-ms timestamp (e.g. `1735509949166`). Contains `api_conversation_history.json` (full LLM request/response history), `ui_messages.json` (rendered chat), `task_metadata.json`, context snapshots.
  - `state/taskHistory.json` — task index in recent versions (older versions kept `taskHistory` in the extension's globalState memento inside the global `state.vscdb`; version-dependent — probe both on capture).
  - `settings/` — MCP settings etc.
  - `checkpoints/` — shadow-git snapshots of workspace files per task; contain absolute paths, machine-specific, restore-risky.
- Portability: tasks are plain JSON; copying `tasks/` + the task index restores history. Task metadata references workspace paths as strings — stale paths on machine B degrade checkpoint/file-context features but conversations remain readable. `[documented/inferred]`
- API keys: VS Code SecretStorage (§6) — not in the JSON files, not portable.

Sources: [cline/cline#7742 (reconstructing task history)](https://github.com/cline/cline/issues/7742), [cline/cline#7101 (taskHistory JSON corruption)](https://github.com/cline/cline/issues/7101), [cline/cline discussion #793](https://github.com/cline/cline/discussions/793), [cline/cline#7929 (history location across IDEs)](https://github.com/cline/cline/issues/7929).

### Roo Code (`rooveterinaryinc.roo-cline`)

Fork of Cline; same shape:

- `globalStorage/rooveterinaryinc.roo-cline/tasks/<taskId>/api_conversation_history.json` + `ui_messages.json`; per-task checkpoints.
- Task history index lives in the extension's **globalState memento** → global `state.vscdb` `ItemTable` under the extension key ([known perf issue RooCodeInc/Roo-Code#3784](https://github.com/RooCodeInc/Roo-Code/issues/3784)) — so a faithful restore must merge that DB key, not just copy files.
- Roo has built-in settings import/export (secrets excluded), which a Backpack adapter could piggyback on.

Sources: [Roo-Code#3784](https://github.com/RooCodeInc/Roo-Code/issues/3784), [Roo-Code#3724 (task history loss)](https://github.com/RooCodeInc/Roo-Code/issues/3724), [community write-up (storage paths)](https://zenn.dev/sunwood_ai_labs/articles/exploring-roo-cline-conversation-history).

---

## 5. Claude Code VS Code extension (`anthropic.claude-code`) `[verified-locally]`

The best case for Backpack: **sessions do not live in VS Code storage at all.** The extension embeds the Claude Code CLI; all conversations are CLI-owned:

- `~/.claude/projects/<munged-project-path>/<sessionUuid>.jsonl` — one JSONL per session; the munge replaces `\`, `/`, `:`, spaces with `-` (e.g. `E:\Projects\session-backpack` → `E--Projects-session-backpack`). 81 project dirs on this machine. Sessions may also have a sibling `<sessionUuid>/` dir. Records carry `sessionId` on every line (`last-prompt`, `mode`, `permission-mode`, `bridge-session`, message records...).
- `~/.claude/history.jsonl` — global prompt history; plus `todos/`, `file-history/`, `plans/` etc.
- Session identity: the session UUID (filename + `sessionId` field). Project association is purely the **munged absolute path** of the dir name.

What the extension keeps in VS Code storage is negligible: global `state.vscdb` key `Anthropic.claude-code` = preferences/experiment gates/announcements only — no conversations. Workspace `state.vscdb` key `agentSessions.model.cache` is VS Code's unified "Agent Sessions" view cache and holds *summaries* of Claude Code (and Codex) sessions — `providerType: "claude-code"`, `resource: "claude-code:/<sessionUuid>"`, label, timing, workingDirectoryPath — a disposable cache rebuilt from `~/.claude`, not a store. `[verified-locally]`

Cross-machine: copy `~/.claude/projects/<dir>` — if machine B uses the **same absolute project path**, sessions appear with zero VS Code surgery. Different path → rename the munged dir to match machine B's path. Auth (`~/.claude/.credentials.json` / OS keychain on macOS, `~/.claude.json` OAuth blocks) must not travel. `[verified-locally + documented]`

(macOS/Linux: same `~/.claude` layout. `[documented]`)

---

## 6. Where secrets live — never portable `[verified-locally]`

VS Code SecretStorage (used by Copilot auth, Cline/Roo API keys, MCP oauth, etc.) is rows in the **global** `state.vscdb` `ItemTable`:

- Keys like `secret://{"extensionId":"<ext>","key":"<name>"}`, `secret://mcpEncryptionKey`, dynamic MCP auth-provider entries.
- Values: `{"type":"Buffer","data":[...]}` — ciphertext from Electron `safeStorage` (Chromium os_crypt):
  - **Windows:** AES-256-GCM key stored in `%APPDATA%\Code\Local State` → `os_crypt.encrypted_key`, itself **DPAPI-wrapped (user + machine bound)**. Verified: `Local State` contains `os_crypt.encrypted_key` on this machine.
  - **macOS:** key in login Keychain ("Code Safe Storage"). `[documented]`
  - **Linux:** libsecret/kwallet, or a hardcoded fallback when no keyring. `[documented]`

Backing up `state.vscdb` therefore carries encrypted secrets that are **undecryptable on machine B** — harmless but dead weight. Backpack must (a) exclude/strip `secret://` keys from anything it syncs, and (b) treat re-authentication on the target machine as an explicit, expected restore step for every extension.

---

## 7. What must travel for machine B to see the same conversations

| Extension | Must travel | Re-key needed on B? | Must NOT travel |
|---|---|---|---|
| Copilot Chat (workspace) | `workspaceStorage/<hash>/chatSessions/*.jsonl`, `chatEditingSessions/**`; merge `chat.ChatSessionStore.index` into target workspace `state.vscdb`; optional `GitHub.copilot-chat/transcripts/`, `chat-session-resources/` | **Yes** — target hash differs per machine (§2) | embeddings caches, `codebase-external.sqlite`, secrets |
| Copilot Chat (empty window) | `globalStorage/emptyWindowChatSessions/*.jsonl` + `chat.ChatSessionStore.index` in **global** `state.vscdb` | No dir re-key; DB merge still required | — |
| Cline | `globalStorage/saoudrizwan.claude-dev/tasks/**`, `state/taskHistory.json` (or globalState key), `settings/` | No (global store) | SecretStorage keys; checkpoints (path-bound, optional) |
| Roo Code | `globalStorage/rooveterinaryinc.roo-cline/tasks/**` + merge extension key in global `state.vscdb` | No (global store); DB merge required | SecretStorage keys |
| Claude Code | `~/.claude/projects/<munged-path>/**` (+ optionally `history.jsonl`, `todos/`) | Only if project path differs → rename munged dir | `~/.claude/.credentials.json`, `~/.claude.json` auth |

---

## 8. Implications for a Backpack adapter

**Capture globs (Windows; substitute roots per OS from §1):**

```
%APPDATA%/Code/User/workspaceStorage/*/workspace.json          # mapping manifest — always capture
%APPDATA%/Code/User/workspaceStorage/*/chatSessions/*.jsonl
%APPDATA%/Code/User/workspaceStorage/*/chatEditingSessions/**
%APPDATA%/Code/User/workspaceStorage/*/GitHub.copilot-chat/transcripts/*.jsonl
%APPDATA%/Code/User/globalStorage/emptyWindowChatSessions/*.jsonl
%APPDATA%/Code/User/globalStorage/saoudrizwan.claude-dev/{tasks,state,settings}/**
%APPDATA%/Code/User/globalStorage/rooveterinaryinc.roo-cline/tasks/**
~/.claude/projects/**  ;  ~/.claude/history.jsonl
```

Plus **extracted** (not raw-copied) `state.vscdb` keys: `chat.ChatSessionStore.index` (per workspace + global), `rooveterinaryinc.roo-cline` / `saoudrizwan.claude-dev` globalState keys. Read via SQLite from a **copied** db file (or immutable/read-only open) — never read the live file naively; WAL means the copy must include `-wal`/`-shm` or use the `.backup` sibling.

**Watch paths:** the `chatSessions/` dirs (JSONL grows during a session — debounce; a session file was seen at 6.8 MB), `tasks/` dirs, `~/.claude/projects/`. Note Copilot appends deltas continuously; capture on window-close/idle rather than per-write.

**Restore steps (Copilot, the hard case):**
1. Require VS Code closed (or at minimum the target workspace's window) — VS Code's in-memory storage cache will clobber external `state.vscdb` writes on exit (§1).
2. Resolve target hash dir: scan `workspaceStorage/*/workspace.json` for the project's folder URI; if absent, create it by computing `md5(fsPath + birthtimeMs)` (§2) or instruct a first open.
3. Copy `chatSessions/` + `chatEditingSessions/` payloads.
4. Merge (don't replace) the `chat.ChatSessionStore.index` JSON in that dir's `state.vscdb`; also write `.backup` sibling or delete it so VS Code doesn't restore a stale copy. Take a pre-write backup of the db.
5. For empty-window chats, same merge against the global db.

**Restore steps (Cline/Roo):** plain file copy into `globalStorage/<ext>/`; for Roo additionally merge the task-history globalState key with VS Code closed. **Claude Code:** copy/rename munged project dirs; no VS Code involvement.

**Risks:**
- Writing SQLite that VS Code holds open → corruption or silent loss; mitigate with "VS Code must be closed" gate (detect running process) + db backup + WAL-aware copy.
- Schema drift: Copilot chat store `version: 3` today; the JSONL kinds and index shape are internal VS Code formats with no stability guarantee — version-stamp captures and validate on restore.
- Remote/WSL profiles live on the other side of the boundary (§1) — out of scope for a first adapter, but users will hit it.
- Cline/Roo checkpoints and task metadata embed absolute paths from machine A; restore conversations, but flag checkpoints as machine-bound.
- Secrets: strip `secret://` rows if ever syncing whole db files; better, never sync `state.vscdb` wholesale — extract/merge keys.

**Adapter difficulty ranking:** Claude Code (trivial, file rename at worst) < Cline (file copy) < Roo (file copy + one DB key) << Copilot Chat (per-workspace re-keying + DB merges + closed-app requirement).
