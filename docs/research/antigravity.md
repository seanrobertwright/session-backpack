# Research: Google Antigravity (IDE + CLI) — Session & State Storage

**Ticket:** [#5](https://github.com/seanrobertwright/session-backpack/issues/5)
**Date:** 2026-08-14
**Method:** Primary evidence is direct inspection of a live Windows 11 installation (Antigravity IDE builds + `agy` CLI v1.1.8). Web research was not available in this session, so macOS/Linux paths are **inferred** from VS Code-fork/Electron conventions and flagged as such. Confidence per claim: `verified-locally` / `inferred` / `unknown`.

---

## 1. Product topology (verified-locally)

Antigravity has two surfaces, and **both keep their agent/session state under `~/.gemini/`** (shared parent dir with the Gemini CLI family), *not* inside the VS Code-fork profile dirs:

| Surface | Binary / install | Agent-state dir |
|---|---|---|
| Antigravity (original IDE channel) | `%LOCALAPPDATA%\Programs\Antigravity`; VS Code fork; user-data at `%APPDATA%\Antigravity`; dataFolder `~/.antigravity` | `~/.gemini/antigravity/` |
| Antigravity IDE (newer channel, v1.107.0; `applicationName: antigravity-ide`, `dataFolderName: .antigravity-ide`) | `%LOCALAPPDATA%\Programs\Antigravity IDE`; user-data at `%APPDATA%\Antigravity IDE` | `~/.gemini/antigravity-ide/` |
| Antigravity CLI (`agy`, internal codename "jetski") | `%LOCALAPPDATA%\agy\bin\agy.exe` | `~/.gemini/antigravity-cli/` |

Also present: `~/.gemini/antigravity-backup/` (migration backup) and `~/.gemini/config/` (shared config: `mcp_config.json`, `hooks.json`, plugins — the IDE's `antigravity/mcp_config.json` is a **symlink** to `~/.gemini/config/mcp_config.json`).

The split matters for Backpack: the VS Code-fork dirs (`%APPDATA%\Antigravity*`) hold editor/UI state and **secrets**; the `~/.gemini/antigravity*` dirs hold the actual conversations/agent state.

## 2. Storage locations per OS

**Windows (verified-locally):**
- Agent/session state: `C:\Users\<user>\.gemini\antigravity\`, `...\antigravity-cli\`, `...\antigravity-ide\`
- IDE profile (VS Code-fork layout): `C:\Users\<user>\AppData\Roaming\Antigravity\User\{settings.json, keybindings.json, mcp.json, globalStorage\state.vscdb, workspaceStorage\<hash>\, History\}` (same layout under `Antigravity IDE`)
- Extensions: `C:\Users\<user>\.antigravity\extensions\` (and `.antigravity-ide` for the new channel)
- CLI binary/updater: `%LOCALAPPDATA%\agy\`

**macOS (inferred, medium confidence):** `~/.gemini/antigravity*/` identical (home-relative, verified pattern); IDE profile at `~/Library/Application Support/Antigravity/` (VS Code-fork convention); extensions `~/.antigravity/`.

**Linux (inferred, medium confidence):** `~/.gemini/antigravity*/`; IDE profile at `~/.config/Antigravity/`; extensions `~/.antigravity/`.

> Open question: confirm macOS/Linux IDE profile dir names ("Antigravity" vs "Antigravity IDE" casing) on real installs.

## 3. Formats & schemas (verified-locally)

### 3.1 Conversations — the core payload
`~/.gemini/<surface>/conversations/` contains one file per conversation, named `<conversation-uuid>.pb` or `<conversation-uuid>.db`:

- **`.pb`** — older format (up to ~May 2026 on this machine): a single binary protobuf blob containing the whole conversation trajectory. Sizes 200 KB–4 MB observed.
- **`.db`** — newer format (from ~July 2026): **SQLite, WAL mode** (live `-wal`/`-shm` siblings observed). One `.pb`+`.db` pair for the same UUID was observed — evidence of an in-place format migration; the app reads both.

SQLite conversation schema (identical in CLI and IDE dbs):

```sql
CREATE TABLE trajectory_meta (trajectory_id text PRIMARY KEY, cascade_id text,
  trajectory_type integer, source integer);
CREATE TABLE steps (idx integer PRIMARY KEY, step_type integer, status integer,
  has_subtrajectory numeric, metadata blob, error_details blob, permissions blob,
  task_details blob, render_info blob, step_payload blob, step_format integer);
CREATE TABLE gen_metadata (idx integer PRIMARY KEY, data blob, size integer);
CREATE TABLE executor_metadata (idx integer PRIMARY KEY, data blob);
CREATE TABLE parent_references (idx integer PRIMARY KEY, data blob);
CREATE TABLE trajectory_metadata_blob (id text DEFAULT "main" PRIMARY KEY, data blob);
CREATE TABLE battle_mode_infos (idx integer PRIMARY KEY, data blob);
```

Step content (`step_payload` etc.) is **protobuf inside SQLite** — opaque without the .proto definitions. `cascade_id` equals the conversation UUID (heritage from Windsurf's "Cascade" agent). `battle_mode_infos`/`battle_id` support the parallel-agents "battle" feature.

### 3.2 Conversation indexes
- **CLI:** `~/.gemini/antigravity-cli/conversation_summaries.db` (SQLite):

```sql
CREATE TABLE conversation_summaries (conversation_id text PRIMARY KEY, title text,
  preview text, step_count integer, last_modified_time datetime, workspace_uris text,
  status text, source text, project_id text, agent_name text,
  parent_conversation_id text, nesting_depth integer, battle_id text,
  winning_conversation_id text, not_fully_idle numeric, killed numeric,
  last_user_input_time datetime, last_user_input_step_index integer,
  app_data_dir text);
```

  Notably `app_data_dir` values include both `antigravity` and `antigravity-cli` — this index spans surfaces; `workspace_uris` is a JSON array of `file:///` URIs with absolute local paths.
- **IDE:** `~/.gemini/antigravity/agyhub_summaries_proto.pb` — binary protobuf index; strings show per-conversation: UUID, title, workspace `file:///` URIs, git remote URL + branch, project UUID (or `outside-of-project`). A copy of trajectory summaries is also cached in the IDE's `state.vscdb` (`antigravityUnifiedStateSync.trajectorySummaries`, base64 protobuf).

### 3.3 Artifacts ("brain")
`brain/<conversation-uuid>/` holds the agent's Artifacts as plain Markdown with revision history:
`task.md`, `implementation_plan.md`, `walkthrough.md`, each with `.metadata.json` and `.resolved`, `.resolved.0..N` revision files. CLI brain dirs contain a `scratch/` subdir with working files the agent created. `brain/tempmediaStorage/` holds media (screenshots/recordings; `bin/webm_encoder.exe` supports browser-session recording).

### 3.4 Other per-surface state (verified-locally)
- `annotations/<conversation-uuid>.pbtxt` — tiny text-protobuf, e.g. `last_user_view_time:{seconds:... nanos:...}` (read-state).
- `antigravity_state.pbtxt` (IDE) / `jetski_state.pbtxt` (CLI) — text protobuf: onboarding progress, `installation_uuid`, last selected model, migration status.
- `implicit/<uuid>.pb` — small protobufs (implicit context per workspace/conversation).
- `knowledge/` — knowledge-base storage (empty here except a lock file).
- `code_tracker/{active,history}`, `context_state/` — code-change tracking / context engine state.
- `user_settings.pb`, `installation_id`, `browserAllowlist.txt`.

### 3.5 CLI-specific files
- `history.jsonl` — prompt history; one JSON object per line: `{"display": <prompt text>, "timestamp": <epoch-ms>, "workspace": <abs path>, "conversationId": <uuid>}` (`conversationId` present on newer lines).
- `cache/last_conversations.json` — map of workspace absolute path → most-recent conversation UUID (**this is the resume pointer**).
- `cache/conversation_metadata.json` — JSON summaries: `{ID, Title, Preview, NumSteps, UpdatedAt, WorkspaceURIs, AppDataDir, ProjectID, AgentName}`.
- `cache/default_project_id.txt`, `settings.json` (permission allowlist, `trustedWorkspaces`, statusLine command), `settings.json.bak`, `import_manifest.json` (records one-time import of MCP servers/commands/skills/hooks from gemini-cli), `plugins/`, `builtin/`, `log/cli-*.log`, `crashes/`, `updater/`.

## 4. Session/conversation identity (verified-locally)

- A session = a **conversation**, identified by a UUIDv4; the same UUID names the conversation file, brain dir, annotation file, and index rows.
- Conversations belong to a **project** (`project_id` UUID, or literal `outside-of-project`); projects group conversations in the Agent Manager UI. (Project name→UUID registry not located as a standalone file; it appears embedded in the protobuf indexes — open question.)
- Conversations are bound to workspaces by **absolute** `file:///` URIs (`workspace_uris`), and CLI resume is keyed by absolute workspace path.
- Sub-agent nesting via `parent_conversation_id` / `nesting_depth`; parallel-model runs via `battle_id` / `winning_conversation_id`.
- Internally a conversation is a **trajectory** (`trajectory_id`, `cascade_id`) of ordered **steps**.

## 5. Resume mechanics

- **CLI (verified data path, inferred behavior):** `agy` in a workspace looks up `cache/last_conversations.json[cwd]` → conversation UUID → opens `conversations/<uuid>.db` and replays steps; `conversation_summaries.db` backs the conversation picker. Exact resume flags/commands not verified.
- **IDE (verified data path, inferred behavior):** Agent Manager lists conversations from `agyhub_summaries_proto.pb` (mirrored into `state.vscdb`), loads full trajectories from `~/.gemini/antigravity/conversations/`. The IDE window/workspace state itself is standard VS Code-fork (`workspaceStorage/<md5-of-folder-uri>/state.vscdb`).
- Cross-surface: the shared schema + `app_data_dir` column suggests CLI and IDE can list each other's conversations. Whether one surface can *resume* the other's conversation is an open question.

## 6. Where secrets live (verified-locally) — CRITICAL for Backpack

- **`%APPDATA%\Antigravity\User\globalStorage\state.vscdb`** (SQLite `ItemTable`) contains:
  - `antigravityAuthStatus` — JSON `{name, apiKey, email, userStatusProtoBinaryBase64}` — an **API key in the clear** (not OS-encrypted) in the state db.
  - `antigravityUnifiedStateSync.oauthToken` — ~1 KB base64 protobuf of OAuth token material.
  - `secret://{extensionId...}` rows — Electron `safeStorage`-encrypted (Windows DPAPI → **machine+user bound**, cannot decrypt elsewhere).
- Chromium layer (`Network\Cookies`, `Local Storage\leveldb`) — os_crypt/DPAPI, machine-bound.
- `~/.gemini/google_accounts.json` — active Google account email only (identity, not a credential).
- **CLI credential storage was not located** (`~/.gemini/antigravity-cli/` has no token file). It may use the IDE's state, an OS keyring, or a file not present here — open question.
- Nothing token-like was found inside `~/.gemini/antigravity*/conversations|brain` themselves, but conversation content can obviously *contain* pasted secrets.

## 7. Portable vs machine-specific

**Portable (content, survives copying):**
- `conversations/*.pb|*.db` (+ `-wal`/`-shm`), `brain/<id>/**`, `annotations/*.pbtxt`
- `conversation_summaries.db`, `agyhub_summaries_proto.pb`, `cache/conversation_metadata.json`
- `history.jsonl`, `settings.json`, `knowledge/`, `~/.gemini/config/mcp_config.json`

**Machine-specific / must NOT travel:**
- `installation_id`, `installation_uuid` (in state pbtxt), `machineid` (IDE profile)
- All of `state.vscdb` auth keys, `secret://` rows (DPAPI-bound), Chromium cookies/caches
- Absolute paths baked into data: `workspace_uris` (`file:///e%3A/Projects/...`), `last_conversations.json` keys, `trustedWorkspaces`, statusLine command paths, MCP server command paths

## 8. What must travel for cross-machine resume (per conversation `<id>`, surface `<S>` ∈ {antigravity, antigravity-cli, antigravity-ide})

1. `~/.gemini/<S>/conversations/<id>.db` (and/or `.pb`) — checkpoint WAL first or copy db+`-wal`+`-shm` together.
2. `~/.gemini/<S>/brain/<id>/**` (artifacts + scratch).
3. `~/.gemini/<S>/annotations/<id>.pbtxt` (optional; read-state only).
4. An index entry so the conversation is listed: CLI — insert/upsert the row in `conversation_summaries.db` (plain SQL, feasible) + entries in `cache/conversation_metadata.json`; IDE — `agyhub_summaries_proto.pb` is an opaque protobuf (merging is risky; whether the app re-scans `conversations/` and rebuilds indexes on startup is an **open question — the single highest-value experiment** for the adapter).
5. Workspace parity: the project must exist at the **same absolute path** on the target machine, or `workspace_uris` (JSON in SQLite — rewritable) and `last_conversations.json` keys (JSON — rewritable) must be rewritten. The IDE protobuf index paths are not safely rewritable.
6. Auth must not travel; the user signs into Antigravity on the target machine first.

## 9. Implications for a Backpack adapter

**Capture globs (per user home):**
```
.gemini/antigravity/conversations/*
.gemini/antigravity/brain/**
.gemini/antigravity/annotations/*
.gemini/antigravity/agyhub_summaries_proto.pb
.gemini/antigravity-cli/conversations/*
.gemini/antigravity-cli/brain/**
.gemini/antigravity-cli/conversation_summaries.db*
.gemini/antigravity-cli/history.jsonl
.gemini/antigravity-cli/cache/*.json
.gemini/antigravity-ide/**   (same sub-globs as antigravity)
```
Optional settings tier: `.gemini/config/mcp_config.json`, `<S>/settings.json` (path-bearing; restore with rewrite). **Exclude**: `%APPDATA%\Antigravity*` entirely (secrets in `state.vscdb`, machine-bound, and huge Chromium caches), `installation_id`, `*_state.pbtxt`, `log/`, `crashes/`, `updater/`, `bin/`.

**Watch paths:** `<S>/conversations/` (WAL churn during active sessions — debounce; capture only when `-wal` quiesces or use SQLite backup API), `<S>/brain/`, `cache/last_conversations.json`.

**Restore steps (draft):** ensure Antigravity installed + signed in → place conversation db/pb + brain + annotations → upsert `conversation_summaries.db` row and `cache/conversation_metadata.json`/`last_conversations.json` with target-machine paths → verify workspace path exists (or rewrite `workspace_uris` via SQL) → launch and test resume.

**Risks:**
- Opaque protobuf payloads and an actively migrating format (`.pb`→`.db` observed) — pin adapter behavior per app version; prefer whole-file copy over surgical edits.
- SQLite WAL: copying mid-write can corrupt; always checkpoint/quiesce.
- IDE index (`agyhub_summaries_proto.pb`) merge risk if the app doesn't rebuild from disk.
- Conversation content itself may contain user secrets → vault encryption is essential (already Backpack's plan).
- Version skew between machines (old-format `.pb` vs new-format reader is handled by the app; reverse direction unknown).

## 10. Open questions

1. Does the IDE/CLI **rebuild summary indexes** from `conversations/` on startup (making restore = "drop files in place")? Needs a live experiment.
2. Where does the **CLI** store its auth token? (No file found under `antigravity-cli/`.)
3. Exact `agy` resume UX/flags; whether IDE can resume a CLI conversation and vice versa (shared index suggests yes for listing).
4. macOS/Linux IDE profile dir names — unverified (inferred `~/Library/Application Support/Antigravity` and `~/.config/Antigravity`).
5. The `.proto` schema for `step_payload` — would unlock partial export/import and redaction; currently opaque.
6. Whether any conversation state lives server-side in Google's backend (Agent Manager "AgyHub" naming hints at a hub service) — if sessions also sync via Google account, Backpack's Antigravity adapter may be lower priority than purely-local tools.
