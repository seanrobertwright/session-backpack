# Research: OpenAI Codex CLI — Session & State Storage

**Ticket:** [#3](https://github.com/seanrobertwright/session-backpack/issues/3)
**Date:** 2026-08-14
**Primary evidence:** live inspection of `C:\Users\seanr\.codex` (codex-cli 0.144.6 installed; rollouts on disk span cli_version 0.42.0 → 0.146.0-alpha). Secondary: openai/codex docs and community documentation.
**Confidence legend:** `[verified-locally]` inspected on this machine · `[documented]` from official/secondary docs · `[inferred]` reasoned from schema/behavior, not directly tested.

---

## 1. Storage locations per OS

Everything lives under a single root: `$CODEX_HOME`, defaulting to `~/.codex` on **all** platforms. `[documented]`

| OS | Default root |
|---|---|
| Windows | `C:\Users\<user>\.codex` `[verified-locally]` — note: a plain dotfolder in the profile root, **not** `%APPDATA%` |
| macOS | `~/.codex` (`/Users/<user>/.codex`) `[documented]` — not `~/Library/Application Support` |
| Linux | `~/.codex` `[documented]` — not XDG (`~/.config`/`~/.local/share`) |

The `CODEX_HOME` environment variable relocates the whole tree. `[documented]` This makes the adapter's path resolution trivial: one root, one env override, identical layout everywhere.

Important nuance: the **VS Code extension, Codex Desktop app, `codex exec`, and the TypeScript SDK all write into the same store**. Rollout files on this machine carry `originator` values `codex_cli_rs`, `codex-tui`, `codex_vscode`, `codex_exec`, `codex_sdk_ts`, `Codex Desktop`, `codex_work_desktop` — all in the same `sessions/` tree. `[verified-locally]` One Backpack adapter covers every Codex surface on a machine.

## 2. Directory inventory (what's in `~/.codex`)

Observed on a heavily-used install (`[verified-locally]` unless noted):

| Path | What it is | Backpack disposition |
|---|---|---|
| `sessions/YYYY/MM/DD/rollout-*.jsonl` | **The sessions.** Full conversation transcripts, one JSONL file per session. 126 files, ~158 MB here (median ~158 KB, max ~52 MB). | **Capture — this is the payload** |
| `history.jsonl` | Cross-session prompt history: one line per user message `{session_id, ts, text}`. Raw prompt text — sensitive. Disabled via `[history] persistence = "none"` `[documented]`. | Capture (optional; merge/append) |
| `session_index.jsonl` | Lightweight index `{id, thread_name, updated_at}` — maps session UUIDs to human names. | Capture (optional) |
| `config.toml` | User config: model, personality, `[features]`, `[mcp_servers.*]`, `[plugins.*]`, `[marketplaces.*]`, notify hooks. | Capture with care — can embed secrets and machine paths (see §6) |
| `auth.json` | **Credentials.** OAuth token set or API key. | **Exclude** (see §6) |
| `AGENTS.md`, `prompts/`, `rules/`, `hooks/`, `hooks.json`, `skills/`, `agents/` | User-authored global instructions, custom prompts, hooks, skills. Portable text. | Capture (nice-to-have "profile" tier) |
| `state_5.sqlite` (+ `-wal`/`-shm`) | Thread index DB: `threads` table with id, **absolute** `rollout_path`, cwd, title, git sha/branch/origin, tokens_used, archived flag, model, etc. Derived from rollouts (has a `backfill_state` table). | Exclude — machine-specific, rebuildable `[inferred]` |
| `sqlite/` (`codex-dev.db`, `goals_1.sqlite`, `memories_1.sqlite`, `logs_2.sqlite`) | Desktop-app thread catalog/automations, per-thread goals, derived memories, logs (63 MB). | Exclude |
| `cache/`, `models_cache.json`, `packages/` (873 MB), `plugins/` (545 MB), `vendor_imports/`, `generated_images/` | Caches, installed plugin/marketplace content, runtimes. Reinstallable. | Exclude |
| `log/`, `sandbox*.log`, `.sandbox/`, `.sandbox-bin/`, `tmp/`, `.tmp/` | Logs, sandbox scratch. | Exclude |
| `installation_id`, `version.json`, `cap_sid`, `.codex-global-state.json` | Per-install identity, update-check state, desktop window state. | Exclude — copying `installation_id` across machines would conflate installs `[inferred]` |
| `external_agent_session_imports.json` | Record of **Claude Code sessions imported into Codex** (`source_path` → `imported_thread_id` + sha256). | Exclude, but interesting: Codex itself already does cross-tool session import |

## 3. Session identity

- A session is identified by a **UUIDv7** (time-ordered). It appears in three places that all agree `[verified-locally]`:
  1. the filename: `rollout-2026-08-01T18-40-30-019fbf7c-6d54-7743-a22c-5fb33767ee95.jsonl`
  2. the first line's `payload.id` / `payload.session_id`
  3. `threads.id` in `state_5.sqlite`
- The filename also embeds the session **start timestamp**, and the file is sharded into `sessions/YYYY/MM/DD/` by start date.
- Sessions may additionally carry a human-assignable **name** (`session_index.jsonl` `thread_name`, `threads.name`); `codex resume`/`archive`/`delete` accept "UUID or session name" `[verified-locally]` (from `codex resume --help`).
- UUIDv7 means filenames are globally unique and sort chronologically — **merging session trees from multiple machines into one vault namespace is collision-free by construction.**

## 4. Rollout file format (the transcript)

JSON Lines; every line is `{"timestamp": "<ISO8601>", "type": "<record type>", "payload": {…}}`. Record types observed `[verified-locally]`:

- **`session_meta`** — first line. Payload: `id`/`session_id`, original `timestamp`, `cwd` (absolute!), `originator`, `cli_version`, `source` (`cli`/`vscode`/…), `thread_source`, `model_provider`, `history_mode`, full `base_instructions.text` (~18 KB of system prompt), and a `git` block `{commit_hash, branch, repository_url}`.
- **`turn_context`** — one per turn. Payload: `turn_id`, `cwd`, `workspace_roots`, `approval_policy`, `sandbox_policy`, `permission_profile`, `model`, `effort`, `personality`, `collaboration_mode`.
- **`response_item`** — the conversation proper. Sub-types seen: `message` (role + content array), `reasoning`, `function_call` / `function_call_output`, `custom_tool_call` / `custom_tool_call_output`.
- **`event_msg`** — UI/telemetry events: `user_message`, `agent_message`, `token_count`, `task_started`, `task_complete`, `thread_settings_applied`, `mcp_tool_call_end`, `web_search_end`, `patch_apply_end`.
- **`world_state`** — snapshot of agent-visible environment/settings state (agents_md text, permissions, skills flags, etc.).

Synthetic (redacted) example of the first two lines:

```jsonl
{"timestamp":"2026-08-01T22:40:30.625Z","type":"session_meta","payload":{"id":"0199…UUID","session_id":"0199…UUID","timestamp":"2026-08-01T22:40:30.625Z","cwd":"c:\\Users\\me\\projects\\demo","originator":"codex_vscode","cli_version":"0.146.0","source":"vscode","thread_source":"user","model_provider":"openai","base_instructions":{"text":"[SYSTEM PROMPT ~18KB]"},"history_mode":"legacy","git":{"commit_hash":"<40-hex>","branch":"main","repository_url":"https://github.com/…"}}}
{"timestamp":"2026-08-01T22:40:41.101Z","type":"event_msg","payload":{"type":"task_started","turn_id":"<uuid>","model_context_window":272000}}
```

Key behavioral fact for sync: **rollout files are append-only, and every resume appends a fresh `session_meta` line** (same id, same original timestamp). A session resumed 8 times here contains 8 identical `session_meta` records interleaved with new turns. `[verified-locally]` So a file's mtime/size change = "session touched", and incremental/append-based diffing works well.

## 5. Resume mechanics

From `codex --help` / `codex resume --help` on 0.144.6 `[verified-locally]`:

- `codex resume` — interactive picker. **Filtered to sessions whose recorded cwd matches the current directory by default**; `--all` shows everything (adds a CWD column).
- `codex resume --last` — continue most recent session, no picker.
- `codex resume <SESSION_ID|name> [PROMPT]` — resume by UUID or session name.
- `codex fork` — branch a copy of a previous session into a new one.
- `codex archive|unarchive|delete <id|name>` — lifecycle management (archived flag lives in the sqlite index; no separate `sessions/archived/` dir here).
- Inside a session, `/rollout` prints the transcript path. `[documented]`

What resume actually does `[inferred from schema + docs]`: it locates the rollout file (via the sqlite `threads.rollout_path` index and/or scanning `sessions/`), replays the `response_item` records to rebuild model context, and appends new records to the **same file**. Community reports confirm the file is required: deleting a rollout orphans its indexed thread. The sqlite index is a derived cache — it has a `backfill_state` watermark table, i.e. it is (re)built by scanning rollouts `[inferred — validate during adapter development]`.

## 6. Where secrets live

1. **`auth.json` — the crown jewels.** `[verified-locally]` Structure (values redacted, lengths real):
   ```json
   {"auth_mode":"chatgpt","OPENAI_API_KEY":null,
    "tokens":{"id_token":"<JWT ~1.9KB>","access_token":"<JWT ~1.8KB>",
              "refresh_token":"<~211 chars>","account_id":"<uuid>"},
    "last_refresh":"<ISO8601>"}
   ```
   In ChatGPT-login mode it holds a **long-lived refresh token**; in API-key mode `OPENAI_API_KEY` is populated instead. Written with 0600 perms on Unix `[documented]`. **Backpack must never sync this file** — even inside an encrypted vault it's a liability, and tokens are trivially re-obtained via `codex login` on the target machine.
2. **`config.toml` can embed secrets.** `[verified-locally]` — MCP server definitions carry API keys in `args` (observed: a hosted-MCP `--key <uuid>` argument) and support `[mcp_servers.X.env]` blocks. If Backpack syncs config, it should either flag/redact `mcp_servers` or treat config as an opt-in "trusted vault only" item.
3. **`history.jsonl` and every rollout file contain raw conversation content** — prompts, file contents read by tools, and any secret that ever appeared in terminal output. Not credentials per se, but the reason the vault must be end-to-end encrypted.
4. Minor: `mcp-oauth-locks/` and MCP OAuth token storage, `cap_sid` — exclude.

## 7. Portable vs machine-specific

**Portable (survives a machine move as-is):**
- `sessions/**/*.jsonl` — content is self-contained; embedded absolute `cwd`/path strings are *metadata and prose*, not live pointers. `[verified-locally]` for structure; cross-machine replay `[inferred]`.
- `history.jsonl`, `session_index.jsonl` (append-mergeable).
- `AGENTS.md`, `prompts/`, `rules/`, `skills/`, `agents/`, `hooks/` (user-authored text).

**Machine-specific (do not sync):**
- `state_5.sqlite` — `threads.rollout_path` and `cwd` are **absolute paths**, on Windows often in `\\?\C:\…` extended form. `[verified-locally]` Rebuildable via backfill.
- All other sqlite DBs, caches, `packages/`, `plugins/`, logs, sandbox dirs, `installation_id`, `version.json`, `.codex-global-state.json` (contains window bounds, workspace-root absolute paths).

**Semi-portable:** `config.toml` — mostly preferences, but observed to contain absolute Windows paths (`notify` command, local marketplace sources, MCP server exe paths) plus possible secrets. Sync only with path/secret awareness, or as a labeled "profile" item the user reviews on restore.

**The cwd problem.** Sessions record the project directory as an absolute path, and the resume picker filters on it. This machine shows 33 distinct cwds across drives (`C:\…`, `E:\…`, OneDrive paths). After restore on a machine where the project lives elsewhere:
- `codex resume <uuid>` and `codex resume --all` still work (id-based / unfiltered) `[verified-locally that the flags exist; behavior documented]`;
- the default cwd-filtered picker won't show the session unless the user is in an identically-named directory;
- new turns execute in the *current* directory, so the user must have the project checked out somewhere — Backpack should restore sessions *alongside* guidance (or metadata) about the original project path/git remote. Conveniently, `session_meta.git.repository_url` + `branch` + `commit_hash` are recorded in line 1 of every rollout — the adapter can surface "this session belongs to repo X @ commit Y" without parsing the whole file.

## 8. What must travel for cross-machine resume

Minimum viable set:
1. `sessions/YYYY/MM/DD/rollout-<ts>-<uuid>.jsonl` — the session(s) to move, restored to the **same relative path** under the target `CODEX_HOME`.
2. That's it. Auth is re-established by `codex login`; the sqlite index backfills; `codex resume <uuid>` (or `resume --all`) finds the file. `[inferred — the single highest-value thing to validate empirically in a spike]`

Quality-of-life additions: `session_index.jsonl` entries (names), `history.jsonl` lines for the moved sessions (up-arrow history), and the user-profile tier (`AGENTS.md`, `prompts/`, config sans secrets).

## 9. Implications for a Backpack adapter

**Capture globs** (relative to `CODEX_HOME`, default `~/.codex`):
```
include:
  sessions/**/rollout-*.jsonl        # tier 1: sessions
  session_index.jsonl                # tier 1
  history.jsonl                      # tier 2 (sensitive: raw prompts)
  AGENTS.md, prompts/**, rules/**, skills/**, agents/**, hooks/**   # tier 3: profile
  config.toml                        # tier 3, secret-scan + path-flag on capture
exclude:
  auth.json, mcp-oauth-locks/**, cap_sid
  *.sqlite, *.sqlite-wal, *.sqlite-shm, sqlite/**, logs_2.*
  cache/**, packages/**, plugins/**, vendor_imports/**, generated_images/**
  log/**, *.log, .sandbox*/**, tmp/**, .tmp/**, node_repl/**, .codex-global-state.json*
  installation_id, version.json, models_cache.json, *.bak
```

**Watch paths:** `sessions/` (recursive) for create + append; `session_index.jsonl` and `history.jsonl` for append. Rollouts grow by whole JSONL lines, so an interrupted copy risks at most one truncated trailing line; capture should either tolerate/trim a partial last line or snapshot on quiesce (file idle N seconds). Files reach tens of MB (52 MB max here) — content-defined chunking or per-file append-diffing pays off for the sync vault.

**Restore steps:**
1. Ensure `CODEX_HOME` exists (create `~/.codex/sessions/YYYY/MM/DD/` as needed).
2. Write rollout files to their original relative paths. Never overwrite an existing file with the same UUID unless vault copy is a strict superset (append-only property makes "longer file wins" a safe merge rule `[inferred]`).
3. Append missing `session_index.jsonl` / `history.jsonl` lines (dedupe by `id` / `(session_id, ts)`).
4. Do **not** touch sqlite DBs; let Codex backfill. Validate in spike: does a foreign rollout appear in `codex resume --all` on first run? (Expected yes.)
5. Prompt user to `codex login` if `auth.json` absent.
6. Surface each session's `git.repository_url` / `branch` / original `cwd` so the user can recreate the working directory before resuming; suggest `codex resume <uuid>`.

**Risks / gotchas:**
- Transcripts are maximally sensitive (code, prompts, anything echoed to the terminal) — E2E encryption of the vault is non-negotiable; never log or preview content server-side.
- Windows path forms (`\\?\C:\…`, mixed-case drive letters) appear inside metadata; treat as opaque strings, never normalize in place.
- Version churn is real: this machine spans rollout schemas from cli 0.42 → 0.146, plus a `history_mode: legacy` → sqlite-threads migration in flight. The adapter should treat rollout JSONL as the durable interface (it has stayed backward-loadable) but pin tests to a version matrix, and re-check when `history_mode` stops saying `legacy`.
- Cross-tool angle: Codex already imports Claude Code sessions (`external_agent_session_imports.json` maps `~/.claude/projects/**.jsonl` → Codex thread ids with sha256). Backpack restoring both tools' stores on one machine keeps that mapping coherent only if both sides travel.
- Desktop/VS Code share the store: capture must not assume the CLI is the only writer; watch for activity whenever any Codex surface is open.
