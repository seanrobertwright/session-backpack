# Research: Claude Code Session & State Storage

**Ticket:** #2 · **Date:** 2026-08-14 · **Evidence:** live inspection of a real Windows install (Claude Code v2.1.232, native install) at `C:\Users\seanr\.claude`, cross-checked against official docs (code.claude.com/docs). Each claim is tagged **[verified-locally]**, **[documented]**, or **[inferred]**.

> Privacy note: all samples below are schema-only or heavily redacted. No conversation content, tokens, or keys appear in this document.

---

## 1. Storage locations per OS

Claude Code keeps *all* user-level state in two places under the user's home directory:

| Item | Windows | macOS / Linux | Notes |
|---|---|---|---|
| Main state dir | `%USERPROFILE%\.claude\` | `~/.claude/` | Overridable via `CLAUDE_CONFIG_DIR` env var [documented] |
| Global config + caches | `%USERPROFILE%\.claude.json` | `~/.claude.json` | Single large JSON file [verified-locally] |
| Project settings | `<repo>\.claude\settings.json`, `settings.local.json`, `CLAUDE.md`, `agents/`, `skills/`, `commands/`, `hooks/` | same | Lives in the repo, travels with git [documented] |
| Project MCP | `<repo>\.mcp.json` | same | [documented] |
| Managed/enterprise settings | `C:\Program Files\ClaudeCode\managed-settings.json` (+ `managed-mcp.json`, `managed-settings.d\`), plus registry `HKLM/HKCU\SOFTWARE\Policies\ClaudeCode` | macOS: `/Library/Application Support/ClaudeCode/…`; Linux: `/etc/claude-code/…` | [documented] |
| OAuth credentials | `~/.claude/.credentials.json` (plaintext file) | macOS: system Keychain, item **"Claude Code-credentials"**; Linux: `.credentials.json` file | Windows/Linux = plaintext JSON [verified-locally on Windows / documented for macOS] |

`CLAUDE_CONFIG_DIR` moves the whole `~/.claude` tree, which is the documented way to relocate storage — relevant if Backpack ever wants to redirect state into a synced folder instead of copying it. [documented]

## 2. Layout of `~/.claude/` (verified on this machine)

```
~/.claude/
├── .credentials.json          # OAuth tokens (SECRET — see §7)
├── .last-cleanup              # ISO timestamp of last retention sweep
├── CLAUDE.md                  # user-global memory (portable)
├── settings.json              # user settings: env, permissions, hooks, model,
│                              #   statusLine, enabledPlugins, effortLevel, …
├── settings.local.json        # local overrides
├── history.jsonl              # GLOBAL prompt history, all projects (see §3)
├── projects/                  # ← THE TRANSCRIPTS (see §3)
│   └── <encoded-cwd>/
│       ├── <session-uuid>.jsonl            # main transcript
│       ├── <session-uuid>.orphaned-*.jsonl # crash/duplicate leftovers
│       ├── <session-uuid>/subagents/       # per-session subagent transcripts
│       │   ├── agent-<17hex>.jsonl
│       │   └── agent-<17hex>.meta.json     # {agentType, description, toolUseId, spawnDepth}
│       └── memory/                          # project-scoped memory dir (auto-created;
│           └── subagents/                   #   MEMORY.md when agent memory is used)
├── tasks/<session-uuid>/N.json  # TodoWrite/task lists, one JSON per task
│                                # {id, subject, description, activeForm, status,
│                                #  blocks[], blockedBy[]} + .lock
├── todos/                       # NOT PRESENT on v2.1.232 — legacy location
│                                # (~/.claude/todos/<sessionId>*.json in 1.x) [documented]
├── shell-snapshots/             # snapshot-bash-<epochMs>-<rand>.sh — captured shell
│                                #   functions/aliases per session launch (machine-specific)
├── session-env/<session-uuid>/  # SessionStart-hook env scripts (sessionstart-hook-N.sh)
├── sessions/<pid>.json          # LIVE session registry: {pid, sessionId, cwd, kind,
│                                #   name, status, bridgeSessionId, …} — runtime only
├── file-history/<session-uuid>/ # checkpointing: <16hex>@vN blobs of edited files
│                                #   (backs /rewind; referenced by file-history-snapshot
│                                #    records in the transcript)
├── teams/<team-name>/config.json# agent-team metadata (members, cwd, leadSessionId)
├── jobs/<8hex>/{state.json,timeline.jsonl}  # background job state
├── plugins/                     # installed plugins, marketplaces, caches
├── agents/  commands/  skills/  hooks/      # user-global custom agents/commands/skills
├── ide/<port>.lock              # IDE extension locks (runtime)
├── statusline*.js, plans/, paste-cache/, debug/, cache/, backups/,
│   daemon.*, chrome/, worktrees/, downloads/, metrics/   # misc runtime/aux
└── settings.json.bak*           # backups made by tooling
```
All of the above **[verified-locally]** except the `todos/` legacy note **[documented]**.

### `~/.claude.json` (global config file)
One big JSON with ~100 top-level keys. The ones that matter for Backpack **[verified-locally]**:

- `projects` — dict keyed by **forward-slash absolute path** (e.g. `"E:/Projects/session-backpack"`). Per-project values: `allowedTools`, `mcpServers` (project-scope servers added via `claude mcp add -s project`… stored here when local-scope), `enabledMcpjsonServers` / `disabledMcpjsonServers`, `hasTrustDialogAccepted`, `lastSessionId`, `lastCost`/`lastDuration`/token counters, `exampleFiles`, `lastSessionMetrics`. **`lastSessionId` is how `--continue` finds the most recent session** (with transcript mtime as the observable fallback). [verified-locally / inferred for exact --continue lookup order]
- `mcpServers` — user-scope MCP server definitions (command/args/env or url). **May contain secrets in `env` and `args`.**
- `oauthAccount` — account metadata (email, org UUID) — identity-bearing.
- `userID`, `machineID`, `anonymousId` — machine/user identifiers.
- Everything else is caches, feature flags, tips counters — safe to drop.

## 3. Transcript format & session identity

### Project-dir encoding
`~/.claude/projects/<project>` where `<project>` = absolute cwd with every non-alphanumeric character replaced by `-`. [documented + verified-locally]
Examples from this machine: `E:\Projects\session-backpack` → `E--Projects-session-backpack`; `C:\Users\seanr\.archon\wt-t2b` → `C--Users-seanr--archon-wt-t2b`. Names >200 chars are truncated + hash-suffixed. [documented]
**Consequence: the encoding is lossy** (`-` vs `_` vs `.` vs `\` all collapse to `-`), so you cannot reliably invert a folder name to a path — but every transcript record carries the real `cwd` field, which is the authoritative decode. [verified-locally]

### Session identity
- A **session ID is a UUIDv4**, which is also the transcript filename: `<session-uuid>.jsonl`. [verified-locally]
- Every line in the file is one JSON object; message records repeat the envelope: `uuid` (per-record UUID), `parentUuid` (previous record — the conversation is a **uuid → parentUuid linked list**), `sessionId`, `cwd`, `version` (CLI version), `gitBranch`, `timestamp`, `isSidechain`, `userType`, `entrypoint`. [verified-locally]
- Record `type`s observed on v2.1.232: `user`, `assistant`, `system`, `attachment` (hook outputs, tool results), `file-history-snapshot` (checkpoint pointers into `file-history/`), plus session-state records: `last-prompt`, `mode`, `permission-mode`, `bridge-session` (links local UUID to a claude.ai bridge session `cse_…` id), `custom-title`, `ai-title`, `agent-name`, `queue-operation`, `summary`/compaction records. [verified-locally]
- **Compaction** writes a `system` record with `subtype: "compact_boundary"`, `logicalParentUuid` (bridging the pre-compact chain), and `compactMetadata: {trigger, preTokens, postTokens, …}`, followed by a `user` record flagged `isCompactSummary: true`. History stays in the file; only the in-context view is replaced. [verified-locally]
- **Subagents** live under `projects/<proj>/<session-uuid>/subagents/agent-<id>.jsonl` + `.meta.json`; sidechain records also appear flagged `isSidechain: true`. [verified-locally]
- The docs explicitly warn the line format is **internal and changes between versions** — a Backpack adapter should treat transcripts as opaque blobs to copy, not parse. [documented]

### Cross-file references
On this version, **resume appends to the same file** — a scan of 700 transcript files found zero files containing more than one `sessionId` and zero message-uuids shared between files. New IDs/files are only created by `/branch` / `--fork-session` (copy-then-diverge) and by crash recovery (`.orphaned-<epoch>-<hex>.jsonl`). Older 1.x builds instead forked a new session ID on every resume with `summary`-type link lines — write-ups describing that are outdated. [verified-locally for 2.1.x / documented for the branching behavior]

### `history.jsonl`
Global prompt-box history (4,264 lines here): `{"display": "<prompt text>", "pastedContents": {}, "timestamp": <epochMs>, "project": "E:\\Projects\\session-backpack", "sessionId": "<uuid>"}`. Contains raw prompt text for **every project** — treat as sensitive; note `project` is a raw absolute path. Per-project `history` arrays inside `.claude.json` are gone on this version. [verified-locally]

## 4. Resume mechanics

- `claude --continue` → most recent session for the current directory (per-project `lastSessionId` in `.claude.json` / newest transcript). [documented + verified-locally that lastSessionId exists per project]
- `claude --resume` → interactive picker listing sessions **for the current worktree** (Ctrl+W widens to all worktrees, Ctrl+A to all projects). [documented]
- `claude --resume <id-or-name>` → since **v2.1.223** searches the current project + its worktrees first, then **every other project on this machine**, so a session can be resumed from a different directory. The cross-project match only resolves if *exactly one* other project holds a transcript with messages for that ID — **a hand-copied duplicate makes resume report not-found** rather than pick arbitrarily. [documented — directly relevant to Backpack restore: don't leave two copies of the same session ID in different project dirs]
- What a resume restores: full history, model, agent, permission mode (never `plan`/`bypassPermissions`), active goal, unexpired scheduled tasks. Not restored: `--mcp-config`/`--settings`/`--add-dir`-style launch flags, background bash tasks. [documented]
- `/cd` relocates a session's transcript to the new directory's project folder (v2.1.169+). [documented]
- Retention: transcripts are deleted after **30 days by default** (`cleanupPeriodDays` in settings.json); `~/.claude/.last-cleanup` timestamps the sweep. **Backpack must back up faster than the reaper.** [documented + marker file verified-locally]

## 5. Portable vs machine-specific state

**Portable (meaningful to sync):**
- `projects/**/*.jsonl` + `projects/**/<session>/subagents/**` — the conversations themselves
- `projects/**/memory/**`, `~/.claude/CLAUDE.md`, project `CLAUDE.md`/`.claude/` (the latter usually travels via git anyway)
- `tasks/<session-uuid>/` — todo lists tied to sessions
- `file-history/<session-uuid>/` — needed if you want `/rewind` checkpoints to survive the move
- `history.jsonl` — prompt history (nice-to-have)
- `settings.json`, `agents/`, `commands/`, `skills/`, `hooks/`, `plugins/` config — environment, not session, but users will want them

**Machine-specific (do not sync, or rewrite on restore):**
- Absolute paths everywhere: project-dir folder names encode the cwd; every transcript record has `cwd`; `history.jsonl` has `project`; `.claude.json` `projects` keys are absolute paths; MCP server commands reference local exe paths. [verified-locally]
- `shell-snapshots/` (captured local shell functions incl. local binary paths), `session-env/`, `sessions/` (live PID registry), `ide/`, `daemon.*`, `jobs/`, `teams/` (contain cwd + tmux pane ids), caches, `debug/`, `paste-cache/`. [verified-locally]
- `machineID`, `userID`, `anonymousId`, `oauthAccount` in `.claude.json`.

**Does a differing project path break resume?** The transcript is found by the *encoded folder name*, not by the `cwd` fields inside it. Two supported paths:
1. **Same path on both machines** — nothing to do; drop the files in and `--resume <id>` works. [inferred from mechanism, high confidence]
2. **Different path** — since v2.1.223, `--resume <session-id>` performs the cross-project search, so a transcript restored under its *original* encoded folder name is still found even though that folder doesn't match any local project — Claude Code resumes it and continues in your current directory. [documented] The conservative adapter move is to **re-encode**: place the `.jsonl` (plus its `<session-uuid>/` subdir and `tasks/`+`file-history/` entries) into the folder for the *new* local path (recompute `<encoded-cwd>`), and ensure the old-path copy is not also present (the exactly-one rule). Stale `cwd` strings inside records don't block resume — each new message written after resume records the new cwd — but tools referencing old absolute paths in earlier turns will confuse the model slightly. [inferred from observed schema + documented duplicate rule]

## 6. Memory

- **User memory**: `~/.claude/CLAUDE.md` — plain markdown, fully portable. [verified-locally]
- **Project memory**: `<repo>/CLAUDE.md` or `<repo>/.claude/CLAUDE.md`, plus `.claude/CLAUDE.local.md` (local-only) — travels with the repo. [documented + verified-locally]
- **Auto memory dirs**: `projects/<encoded-cwd>/memory/` (with `subagents/` inside; `MEMORY.md` appears when the agent-memory feature writes) — portable but keyed to the encoded path, so it must be re-encoded on path change like transcripts. [verified-locally]

## 7. Secrets & credentials — the danger list

| Location | Contents | OS |
|---|---|---|
| `~/.claude/.credentials.json` | `claudeAiOauth.accessToken` + `refreshToken` (+ expiry, scopes, subscription tier), `mcpOAuth.<server>.accessToken` + OAuth client info, `organizationUuid` | Windows & Linux — **plaintext file** [verified-locally on Windows] |
| macOS Keychain item "Claude Code-credentials" | same OAuth payload | macOS [documented] |
| `~/.claude.json` → `mcpServers[*].env` / `args` | frequently API keys/tokens for MCP servers | all |
| `~/.claude/settings.json` → `env` | user-set env vars, may include keys | all |
| `history.jsonl`, transcripts | anything the user ever pasted (keys, URLs, internal data) | all |
| `.claude/settings.local.json`, project `.mcp.json` env blocks | same class of risk | all |

**Backpack should exclude `.credentials.json` by default** (auth is per-machine and re-login is cheap; syncing refresh tokens between machines multiplies theft surface and can cause refresh races) and treat everything else it *does* capture as secret-bearing — hence the vault must be encrypted, which matches Backpack's design. [verified-locally + inferred recommendation]

## 8. Minimum travel set for cross-machine resume

For one session `<sid>` from project at `<path>` (encoded `<enc>`):

1. `~/.claude/projects/<enc>/<sid>.jsonl` — **required**
2. `~/.claude/projects/<enc>/<sid>/` (subagents) — required if subagents were used and you want their transcripts intact
3. `~/.claude/tasks/<sid>/` — todo list continuity
4. `~/.claude/file-history/<sid>/` — `/rewind` checkpoints (optional; without it resume works but rewind history is gone)
5. `~/.claude/projects/<enc>/memory/` — project auto-memory (session-adjacent, sync once per project)
6. The project working tree itself (git) — Claude's context assumes the files it was editing exist; Backpack should record the repo remote + branch + dirty-file status alongside, or the resumed session hallucinates against missing files. [inferred]

*Not* required: shell-snapshots, session-env, sessions/, credentials, `.claude.json` (a fresh machine regenerates it; only per-project `allowedTools`/trust flags are lost, which merely re-prompts). Version skew matters: the record format is version-internal, so resuming a transcript written by a much newer CLI on an older CLI is unsupported. [documented]

## 9. Implications for a Backpack adapter

**Capture globs** (relative to `~/.claude`, i.e. `CLAUDE_CONFIG_DIR`):
```
projects/**            # transcripts, subagents, memory  (core)
tasks/**               # todos
file-history/**        # checkpoints (size-heavy; make optional tier)
history.jsonl          # prompt history (optional tier)
CLAUDE.md              # user memory
settings.json, agents/**, commands/**, skills/**, hooks/**   # "environment" tier, opt-in
```
**Default-exclude:** `.credentials.json`, `shell-snapshots/`, `session-env/`, `sessions/`, `ide/`, `daemon*`, `jobs/`, `cache/`, `debug/`, `paste-cache/`, `plugins/cache/`, `*.bak*`, `.claude.json` (or capture only its `projects` sub-map, filtered).

**Watch paths for auto-backup:** watch `projects/**` for `*.jsonl` appends — Claude Code writes the transcript continuously (line-append), so debounced (~5–30s) copy-on-change is safe; JSONL is crash-tolerant (a torn final line loses one record, not the file). Also watch `tasks/**` and `file-history/**`. The `sessions/<pid>.json` registry is a convenient "is a session live right now" signal (`status: idle/running`) for choosing quiet moments. [verified-locally]

**Restore steps:**
1. Recompute `<enc>` from the *destination* project path; write transcript + subagent dir there (rewrite the folder name, not file contents).
2. If destination path == origin path, plain copy.
3. Never leave the same `<sid>.jsonl` under two different `<enc>` folders on one machine (breaks cross-project resume resolution).
4. Restore `tasks/<sid>` and `file-history/<sid>` verbatim (session-uuid-keyed, path-free).
5. Do not restore `.claude.json`; at most merge per-project `allowedTools`/`hasTrustDialogAccepted` for the destination path key.
6. Tell the user: `claude --resume <sid>` (or `--continue` from the project dir).

**Risks / gotchas:**
- 30-day auto-cleanup deletes transcripts — backup cadence must beat `cleanupPeriodDays`.
- Format is explicitly version-internal; adapter must be copy-based, never parse-based, and should record the `version` field for compat warnings.
- OneDrive-style sync of `~/.claude` wholesale is dangerous (live-file contention, plaintext credentials in the sync folder) — validates Backpack's copy-into-encrypted-vault model.
- Path-encoding is lossy; always derive mapping from a transcript record's `cwd` field, not from folder names.
- Concurrent resume of one session from two machines would interleave/conflict — Backpack needs last-writer-wins or lockout semantics per session.
- Desktop app / VS Code / claude.ai ("bridge") sessions keep separate histories; `bridge-session` records tie a local UUID to a `cse_…` cloud id — cloud-side state is out of Backpack's reach.

---

### Sources
- Local inspection: `C:\Users\seanr\.claude`, `C:\Users\seanr\.claude.json` (Claude Code v2.1.232, Windows 11, native install)
- [Manage sessions — Claude Code docs](https://code.claude.com/docs/en/sessions)
- [Settings — Claude Code docs](https://code.claude.com/docs/en/settings)
- [Where does Claude Code store conversation history (LLMnesia)](https://www.llmnesia.com/blog/where-does-claude-code-history)
- [Claude Code SSH keychain fix (macOS Keychain item name)](https://oldeucryptoboi.com/blog/claude-code-ssh-keychain-fix/)
