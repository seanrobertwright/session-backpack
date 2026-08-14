# Research: pi & oh-my-pi — session storage, state, and cross-machine resume

Ticket: [#7](https://github.com/seanrobertwright/session-backpack/issues/7)
Date: 2026-08-14
Method: primary evidence from live installs on the Windows dev machine (`pi` v0.84.2 via npm global, `oh-my-pi` v17.3.2 via bun global), cross-checked against bundled official docs (`docs/sessions.md`, `docs/session-format.md`, `docs/environment-variables.md`, `docs/providers.md` shipped inside the pi package) and both GitHub repos. Confidence is tagged per claim: **[verified-locally]**, **[documented]**, **[inferred]**.

---

## 1. Relationship between pi and oh-my-pi

- **pi** is the coding agent from the **pi-mono** monorepo by Mario Zechner (badlogic). The repo now lives under his company org — `github.com/earendil-works/pi-mono` (badlogic/pi-mono redirects) — and the CLI ships as npm package `@earendil-works/pi-coding-agent`, binary `pi`. **[verified-locally + documented]**
- **oh-my-pi** is a **hard fork** of pi by Can Bölük (`github.com/can1357/oh-my-pi`, homepage `omp.sh`), package `@oh-my-pi/pi-coding-agent`, binary `omp`. Its package.json declares author "Can Boluk" with Mario Zechner as contributor and describes itself as "a fork of Pi ... that adds everything you're missing" (IDE/LSP integration, subagents, memory backends, session checkpoint/rewind, importers). **[verified-locally + documented]**
- They are **separate installs with separate state roots** (`~/.pi` vs `~/.omp`) and **do not share state at runtime**, but the **session file format is shared lineage**: both write JSONL session files with the same header (`type:"session"`, `version:3`) and the same entry-tree model (`id`/`parentId`). oh-my-pi's format is a superset (extra entry types and header fields — see §4). **[verified-locally]**
- Version numbering diverged completely (pi 0.84.x vs omp 17.x) — do not treat omp as "pi vNext"; treat them as **two tools with a common ancestor format**. **[verified-locally]**

## 2. Storage locations

### pi

| Item | Path (Windows, verified) | Notes |
|---|---|---|
| State root | `C:\Users\<user>\.pi\agent\` | everything lives under `<home>/.pi/agent` |
| Sessions | `~/.pi/agent/sessions/<encoded-cwd>/<timestamp>_<uuid>.jsonl` | one dir per project cwd |
| Settings | `~/.pi/agent/settings.json` | JSON: default model/provider, theme, installed packages list |
| Auth / secrets | `~/.pi/agent/auth.json` | **plaintext** OAuth tokens + API keys (see §7) |
| Custom providers | `~/.pi/agent/models.json` | user-defined providers (may embed apiKey/baseUrl, e.g. local ollama) |
| Model catalog cache | `~/.pi/agent/models-store.json` | per-provider catalog cache (`checkedAt`, `etag`, `models`) — no secrets |
| Project trust | `~/.pi/agent/trust.json` | map of absolute project paths -> trusted bool |
| Extensions/packages | `~/.pi/agent/npm/` (~100 MB node_modules), `~/.pi/agent/extensions/`, `~/.pi/agent/skills/` | reinstallable from `settings.json` `packages` list |
| Helper binaries | `~/.pi/agent/bin/` (rg.exe, fd.exe) | machine/arch-specific, re-downloaded |
| Temp | `~/.pi/agent/tmp/` | scratch |
| Project-local | `<project>/.pi/` (settings.json, npm/, extensions) | travels with the repo, not the vault |

- macOS/Linux: same layout under `~/.pi/agent` — path is `os.homedir()`-based, no XDG/AppData indirection. **[documented]** (env-vars doc: `PI_CODING_AGENT_DIR` overrides, default `~/.pi/agent`).
- Overrides: `PI_CODING_AGENT_DIR` (whole agent dir). **[documented]**

### oh-my-pi

| Item | Path (Windows, verified) | Notes |
|---|---|---|
| Root | `C:\Users\<user>\.omp\` | agent state in `~/.omp/agent\` |
| Sessions | `~/.omp/agent/sessions/<encoded-cwd>/<timestamp>_<uuid>.jsonl` | same scheme as pi; plus optional sibling dir `<timestamp>_<uuid>/` holding per-entry artifacts (e.g. `8.bash-original.log` = untruncated tool output, numbered by entry) |
| Config | `~/.omp/agent/config.yml` | YAML (shellPath, modelRoles, theme, setupVersion); a `settings.json.bak` shows migration from pi-style settings.json |
| Auth / secrets + misc | `~/.omp/agent/agent.db` (SQLite, WAL) | tables: `auth_credentials` (**plaintext token JSON** in `data` column), `auth_credential_blocks/leases`, `usage_history`, `usage_cost_history`, `client_usage`, `clients` (install_id+hostname), `cache`, `settings`, `meta`, `model_perf` |
| Prompt history | `~/.omp/agent/history.db` (SQLite) | `history(prompt, created_at, cwd, session_id)` + FTS5 index |
| Model catalog cache | `~/.omp/agent/models.db` (SQLite) | `model_cache` per provider — cache only |
| Custom providers | `~/.omp/agent/models.yml` | documented; not present on this machine |
| Terminal mapping | `~/.omp/agent/terminal-sessions/wt-<guid>` | maps terminal window -> session (empty marker files here) |
| Install identity | `~/.omp/install-id` | per-machine UUID |
| Logs / runtime | `~/.omp/logs/`, `~/.omp/run/daemons/<id>/` (broker.pid, broker.token), `~/.omp/gpu_cache.json`, `~/.omp/natives/<ver>/*.node` | machine-specific, never back up |
| Profiles | `~/.omp/profiles/<name>/agent/` | `--profile` / `OMP_PROFILE` isolates auth+sessions+settings **[verified in source]** |
| Project-local | `<project>/.omp/` | project agents/config |

- macOS/Linux: default is the same `~/.omp/agent`. On Linux/macOS, **opt-in** XDG split (`omp config init-xdg`) relocates state to `$XDG_DATA_HOME/omp`, `$XDG_STATE_HOME/omp`, `$XDG_CACHE_HOME/omp` — an adapter must probe both. **[verified in source, `pi-utils/src/dirs.ts`]**
- Overrides: `PI_CONFIG_DIR` (dir *name*, default `.omp`), `PI_CODING_AGENT_DIR` (kept for pi compat — "Session storage directory, default `~/.omp/agent`"), `--session-dir`, `OMP_PROFILE`/`PI_PROFILE`. **[verified-locally, `omp --help` + source]**

### Session directory name encoding (both tools)

The per-project dir name is the absolute cwd with separators replaced: `E:\Projects\second-brain` -> `--E--Projects-second-brain--`, `/` and `\` -> `-`, prefixed/suffixed with `--`. **[verified-locally]** This means the dir name is **machine-path-specific** (see §8).

## 3. Session identity

- Session ID = **UUIDv7** (time-ordered, e.g. `019ff22d-65fa-7...`). Filename = `<ISO-8601 timestamp with dashes>_<uuid>.jsonl`; the same id is in the header line. **[verified-locally]**
- Resume accepts full/partial ID or file path; IDs are globally unique so files can be relocated. **[documented]**
- omp additionally has a per-machine `install-id` and a `clients` table (install_id, hostname) used for usage aggregation — identity of the *machine*, not the session. **[verified-locally]**

## 4. Session file format

Both: JSONL, one JSON object per line, `type` discriminator, entries form a **tree** via `id`/`parentId` (in-place branching; the "active leaf" defines the current conversation path). Format version history: v1 linear -> v2 tree -> v3 renamed `hookMessage` to `custom`; old files auto-migrate on load. **[documented + verified-locally: all local files are v3]**

pi entry types observed: `session` (header: type, version, id, timestamp, cwd), `model_change` (provider, modelId), `thinking_level_change`, `message` (`message: {role, content[], timestamp, ...}` — roles user/assistant/toolResult with typed content blocks: text, image(base64), thinking, toolCall), `custom` (extension data). **[verified-locally, matches session-format.md]**

oh-my-pi additions **[verified-locally]**:
- line 0 is a fixed-width `title` record (`{type:"title", title, source, updatedAt, pad, v}` — padded for in-place rewrite so pickers can read titles without scanning);
- header gains `title`, `titleSource`;
- extra entry types: `title_change`, `credential_pin` (`provider` + credential `hash` — records which credential the session used);
- `model_change` uses a single `model` field (+`resolvedModelIsFallback`) instead of pi's `provider`+`modelId` pair;
- large tool outputs are spilled to a sibling `<session-id>/` artifact directory instead of living only inline.

Compatibility verdict: pi sessions are readable by omp (same core schema; omp evolved from it) — **[inferred, high confidence]**; omp sessions read back into pi would hit unknown entry types/fields — **[inferred]** treat as one-way. omp also ships `--from-claude` and `--from-codex` session importers. **[verified-locally, --help]**

## 5. Resume mechanics

pi **[documented]**: `pi -c` (continue most recent session *for the current cwd*), `pi -r` (picker, scoped to current project dir), `pi --session <path|id>`, `pi --fork <path|id>`, `pi --no-session`, `--name`; in-TUI `/resume`, `/tree` (branch navigation), `/fork`, `/clone`, `/compact`.

oh-my-pi **[verified-locally]**: `omp -c`, `omp -r [id-prefix|path|picker]`, `--session-dir <dir>`, `--no-session`, `--profile <name>`, plus checkpoint/rewind session ops. Discovery of "most recent for this project" goes through the encoded-cwd directory, same as pi.

Key implication: **resume discovery is keyed on the absolute cwd**. A session restored to a machine where the project lives at a different absolute path will not be found by `-c`/`-r` unless it is placed in (or the header rewritten for) the *new* path's encoded directory. Explicit `--session <file>` bypasses this. **[verified-locally (dir scheme) + inferred (behavior)]**

## 6. Portable vs machine-specific state

Portable (worth backing up):
- `sessions/**/*.jsonl` (+ omp's sibling artifact dirs) — the conversations themselves. Content is self-contained (messages, tool calls/results, base64 images inline).
- pi `settings.json` (includes the `packages` list = recipe to reinstall extensions), `models.json`, `trust.json` (paths are absolute — portable only between machines with identical paths).
- omp `config.yml`, `models.yml`, `history.db` (prompt history is portable and useful).

Machine-specific (never sync): `~/.pi/agent/npm/`, `bin/`, `tmp/`; `~/.omp/natives/`, `run/`, `logs/`, `gpu_cache.json`, `install-id`, `terminal-sessions/`, `models.db`/`models-store.json` (caches), `agent.db-wal/-shm` sidecars, usage/cost tables (arguably portable but low value and entangled with secrets in the same DB).

Embedded machine-specifics inside portable files: absolute `cwd` in every session header and encoded dir names; absolute `shellPath` in omp `config.yml`; absolute paths in `trust.json`. **[verified-locally]**

## 7. Where secrets live (critical for the vault)

- **pi: `~/.pi/agent/auth.json`** — plaintext JSON. Verified shape (values redacted): per-provider objects, e.g. `anthropic: {type, access, refresh, expires}` (OAuth access+refresh tokens), `openai-codex: {..., accountId}`, `github-copilot: {...}`, `openrouter: {type, key}` (API key). `models.json` can also embed API keys for custom providers. **[verified-locally]**
- **oh-my-pi: `agent.db` -> `auth_credentials` table** — `data` column holds **plaintext JSON** per credential: oauth rows with `access`, `refresh`, `expires`, `email`, `accountId`, `orgId`, `orgName`; api_key rows with `key`. Secrets are *inside the same SQLite file* as settings/usage — you cannot copy agent.db without copying tokens. Also `~/.omp/run/daemons/*/broker.token` (runtime auth broker token). **[verified-locally]**
- Sessions contain conversation text and full tool output (may incidentally contain secrets the user pasted or that commands printed) but no structured credentials; omp's `credential_pin` stores only a hash. **[verified-locally]**

## 8. What must travel for cross-machine resume

Minimum viable set:
1. The session `.jsonl` file(s) — and for omp, the sibling `<session-id>/` artifact dir if present.
2. Placement under the **target machine's** encoded-cwd directory for the project (recompute `--<encoded path>--` from the destination project path), or the user must resume with an explicit `--session <file>`.
3. Working credentials on the target machine (auth.json / auth_credentials) — but these should be *re-established locally* (login) or synced through a separately-encrypted secrets channel, never as plain vault files.

Optional but high-value: pi `settings.json` (drives extension reinstall on first run), omp `config.yml` (strip/remap `shellPath`), omp `history.db`, custom `models.json`/`models.yml` (redact embedded keys or treat as secret-class).

Not needed: npm/, bin/, natives, caches, logs, run/, install-id (must *not* travel — it is the per-machine identity), terminal-sessions.

## 9. Backpack adapter implications

**One adapter can serve both tools** as a parameterized family: same sessions layout (`<root>/agent/sessions/<encoded-cwd>/<ts>_<uuidv7>.jsonl`), same tree JSONL core, same env override (`PI_CODING_AGENT_DIR`). Differences are config-file shape (json vs yml+sqlite), omp's artifact sidecar dirs, omp profiles, and Linux XDG opt-in. Recommend a shared `pi-family` adapter with two tool descriptors. **[verified-locally]**

Capture globs (per home dir; omp repeated for each `profiles/<name>`):

```
# pi
.pi/agent/sessions/**/*.jsonl
.pi/agent/settings.json
.pi/agent/models.json          # secret-class if custom keys present
.pi/agent/trust.json
# oh-my-pi (also under .omp/profiles/*/agent/)
.omp/agent/sessions/**              # includes artifact sidecar dirs
.omp/agent/config.yml
.omp/agent/models.yml
.omp/agent/history.db
# EXCLUDE always
.pi/agent/{npm,bin,tmp,extensions}/**
.pi/agent/models-store.json
.omp/agent/{agent.db*,models.db*,terminal-sessions}/**
.omp/{logs,run,natives,gpu_cache.json,install-id}
# NEVER capture into the plain vault
.pi/agent/auth.json
.omp/agent/agent.db            # contains auth_credentials
```

Watch paths: `~/.pi/agent/sessions/` and `~/.omp/agent/sessions/` (recursive). JSONL files are append-mostly (omp rewrites the padded title line in place); WAL-mode SQLite (`history.db`) needs snapshot-on-quiesce (`sqlite3 .backup` or copy db+wal+shm atomically), not naive file watch. Linux: also probe `$XDG_DATA_HOME/omp` / `$XDG_STATE_HOME/omp`.

Restore steps:
1. Resolve target root (respect `PI_CONFIG_DIR`/`PI_CODING_AGENT_DIR`/profile).
2. For each captured session, recompute the encoded-cwd dir from the *destination* project path (adapter needs the project-path mapping Backpack already tracks); optionally rewrite the header `cwd` field to match. Unmapped sessions can be restored verbatim — resume then requires `--session <path>`.
3. Copy `.jsonl` + artifact dir; merge by session UUID (UUIDv7 = no collisions; newer file wins by mtime/size, or line-count union is possible since entries are append-only).
4. Restore config files; remap/drop `shellPath` and other absolute paths; drop `trust.json` entries for paths that don't exist.
5. Tell the user to run `/login` per provider (or restore secrets via the encrypted secrets channel into `auth.json`; for omp there is no supported file to drop — writing `auth_credentials` rows into agent.db is schema-versioned and fragile, so prefer re-login).

Risks:
- **Secrets adjacency**: omp fuses secrets and settings in one SQLite file; a lazy "grab ~/.omp/agent" capture exfiltrates refresh tokens into the vault. The adapter must be deny-by-default on `agent.db` and `auth.json`.
- **Path remapping**: cwd-encoded dir names + absolute `cwd` headers mean naive restore breaks `-c`/`-r` discovery across machines with different project roots (Windows `E:\...` vs mac `/Users/...`).
- **Format drift**: omp releases fast (17.x, schema-versioned DBs, session v3+extensions); pin adapter behavior to header `version` and ignore unknown entry types.
- **Live-write races**: capture during an active session can catch a half-written JSONL line (append) or mid-rewrite title line; snapshot on file-quiesce debounce.
- **Big sessions**: base64 images inline and large artifact sidecars — dedupe/chunk in the vault.

## 10. Confidence summary

Verified locally: install identities and versions, all Windows paths, JSONL schemas (both), SQLite schemas (all three omp DBs), auth storage shapes (keys only), CLI resume flags (omp `--help`, pi docs), profile/XDG/env resolution (read from shipped source). Documented: pi resume flags/commands, session version history, `PI_CODING_AGENT_DIR`, fork relationship (both READMEs). Inferred: pi->omp one-way session compatibility; exact resume-discovery failure mode on unmapped cwd (follows directly from the verified dir-encoding scheme but not tested end-to-end).
