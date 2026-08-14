# Research: Tauri v2 Stack for Backpack

Resolves [#9](https://github.com/seanrobertwright/session-backpack/issues/9). Researched 2026-08-14. Confidence markers: **[High]** = verified against docs/crates.io; **[Medium]** = multiple secondary sources or well-established knowledge; **[Low]** = inference, verify during build.

## Recommended stack (summary)

| Area | Recommendation | Version (verified 2026-08) | Runner-up |
|---|---|---|---|
| Shell | Tauri v2 | 2.x (stable; v2.10.x current) | — |
| File watching | `notify` + `notify-debouncer-full` | 8.2.0 / 0.7.0 stable (9.0-rc / 0.8-rc in flight) | `watchexec` lib (heavier) |
| Tray + background | Tauri built-in `tray-icon` feature + `tauri-plugin-autostart` | in-tree / plugins-workspace v2 | — |
| Vault encryption | `age` crate, passphrase (scrypt) recipient | age 0.12.1 | `chacha20poly1305` 0.10 + `argon2` 0.5 (custom format) |
| Passphrase caching | `keyring` crate (OS keychain), opt-in | keyring 4.1.6 | tauri-plugin-stronghold (not recommended) |
| Index DB | `rusqlite`, `bundled` + FTS5 | rusqlite 0.40.2 | `sqlx` 0.9 (async, heavier) |
| Updater | `tauri-plugin-updater` + static JSON on GitHub Releases | plugins-workspace v2 | CrabNebula Cloud |
| Installers | NSIS (Win), .dmg + notarized .app (macOS), AppImage + .deb (Linux) | Tauri bundler | MSI |
| Win signing | Azure Trusted/Artifact Signing | $9.99/mo Basic | OV cert (~$200–400/yr) |
| Frontend | Svelte 5 + Tailwind CSS v4 + CSS custom-property theming | — | SolidJS; React if ecosystem needed |
| IPC | Commands + `Channel<T>` for progress streams; events for broadcast | Tauri v2 core | — |

---

## 1. File watching: `notify` + `notify-debouncer-full`

**Recommendation:** `notify` 8.2.0 (stable, Aug 2025) with `notify-debouncer-full` 0.7.0. A 9.0/0.8 release-candidate line exists (rc.4, May 2026) — start on stable, plan a minor migration. **[High — crates.io]**

- `notify` is the de-facto standard cross-platform watcher: inotify (Linux), FSEvents (macOS), ReadDirectoryChangesW (Windows), plus a `PollWatcher` fallback. **[High]** ([docs.rs/notify](https://docs.rs/notify/latest/notify/), [crates.io](https://crates.io/crates/notify))
- **Debouncing is mandatory for Backpack's use case.** Agent CLIs (Claude Code, etc.) append to session JSONL files continuously; a single save can emit `Modify, Modify, Create, Modify` bursts. `notify-debouncer-full` coalesces bursts, tracks renames via a file-ID cache, and emits a clean event after a quiet period ([docs.rs/notify-debouncer-full](https://docs.rs/notify-debouncer-full/latest/notify_debouncer_full/)). Use a generous debounce (2–5 s) since sessions are written for the lifetime of an agent run — Backpack wants "session settled", not keystroke granularity. Consider a second application-level quiescence timer (e.g., back up only after N seconds of no events on a session file). **[High for crate behavior; Medium for tuning numbers]**
- **Gotchas watching dirs other apps write rapidly** **[Medium–High]**:
  - Windows `ReadDirectoryChangesW` has a fixed kernel buffer; heavy event storms can overflow it and silently drop events. Treat the watcher as a hint, and pair it with a periodic full rescan (mtime/size diff) so nothing is missed.
  - Linux inotify has per-user watch limits (`fs.inotify.max_user_watches`, often 8192/65536); watching deep trees (e.g., `~/.claude/projects/**`) can exhaust them. Watch the top-level session dirs non-recursively where possible, or handle `ENOSPC` gracefully.
  - macOS FSEvents coalesces aggressively and can deliver directory-level rather than file-level precision; rescan the reported directory.
  - Files may still be mid-write when the event fires: copy with retry/backoff, and verify size+hash stability before treating a snapshot as durable.
- **Cloud-synced folders (the vault side)**: notify explicitly documents that network/virtual filesystems may emit no events; PollWatcher is the workaround ([docs.rs known problems](https://docs.rs/notify/latest/notify/#known-problems)). Additional real-world issues: OneDrive/Dropbox "Files On-Demand" placeholder files can appear as create+modify storms or as 0-byte placeholders until hydrated; sync engines rewrite files via temp-file+rename which changes inodes/file-IDs; watchers on Windows sometimes silently die on synced folders (seen in Syncthing's tracker: [forum thread](https://forum.syncthing.net/t/file-watcher-on-windows-sometimes-stops-working-for-specific-folders/17194)). **Design decision this implies: Backpack should not need to watch the vault folder at all** — it *writes* to the vault and *scans* it on demand/interval (index rebuild), and only *watches* the local agent session directories, which are ordinary local dirs. That sidesteps the whole class of cloud-sync watching problems. **[Medium confidence on individual behaviors; High confidence on the design implication]**
- **Alternative:** the `watchexec` library crate layers robustness (process-aware filtering, restarts) over notify but drags in a large dependency tree; overkill here. **[Medium]**

## 2. System tray + autostart (backups while minimized)

**Recommendation:** Tauri v2 built-in tray (`tray-icon` Cargo feature on the `tauri` crate — it is core in v2, not a separate plugin) + official `tauri-plugin-autostart`. **[High]**

- Tray: `TrayIconBuilder` in Rust with menu, tooltip, click events, runtime icon swaps (e.g., animated/badged icon during backup). Docs: [v2.tauri.app/learn/system-tray](https://v2.tauri.app/learn/system-tray/). Known wart: hot-reload in dev can duplicate tray icons ([tauri#8982](https://github.com/tauri-apps/tauri/issues/8982)) — build the tray in `setup()` once, keep a handle. **[High]**
- Keep-alive pattern: intercept `WindowEvent::CloseRequested`, call `api.prevent_close()` and hide the window instead; app keeps running with the watcher active. On macOS set `ActivationPolicy::Accessory` when hidden so no Dock icon lingers. **[High — standard documented pattern]**
- Autostart: `tauri-plugin-autostart` (official, plugins-workspace) handles registry Run key (Win), LaunchAgent (macOS), and .desktop autostart entry (Linux). Launch with a `--minimized` flag argument so boot-time start goes straight to tray. Docs: [v2.tauri.app/plugin/autostart](https://v2.tauri.app/plugin/autostart/). Caveats: MacLaunchAgent-based autostart shows in macOS Login Items (good, user-visible); on Linux, `~/.config/autostart` works on major desktops but not guaranteed on minimal WMs. **[High for plugin existence/mechanics; Medium for Linux edge cases]**
- Honest tradeoff: "runs while minimized" is app-lifetime only. If the user quits the app or never logs in, no backups happen. A true daemon/service is out of scope for v1 and would hurt the "simple install" goal; autostart-at-login is the right 90% solution. **[Medium — judgment]**

## 3. Vault encryption: age vs. hand-rolled AEAD; keychain storage

**Recommendation:** `age` crate (0.12.1, Jul 2026) with `Encryptor::with_user_passphrase` (scrypt-based passphrase recipient). Store the passphrase optionally in the OS keychain via `keyring` 4.1.6 for convenience. **[High on versions; High on recommendation]**

- **Why age wins for Backpack:**
  - It is a *format*, not just a cipher: versioned header, scrypt work-factor encoding, chunked STREAM (ChaCha20-Poly1305) payload with per-chunk authentication — i.e., the tamper-detection, nonce-management, and chunking work you'd otherwise hand-build. ([age spec](https://age-encryption.org/v1), [docs.rs/age](https://docs.rs/age))
  - **Interop is a genuine recovery story:** any vault file can be decrypted with the `age`/`rage` CLI on any machine, even if Backpack is dead/abandoned. For a backup product, "your data is not held hostage by our app" is a headline feature. **[High]**
  - Maintained by Filippo Valsorda (Go reference) with the Rust crate powering `rage`; crate is labeled "[BETA]" but the format is stable and widely deployed (passage, SOPS, etc.). **[Medium–High]**
  - Future-proof: age supports X25519 recipients, so a later "recovery key file" feature (encrypt to passphrase AND a printed recovery key) is a format-native extension, not a redesign. **[High]**
- **Alternative — `chacha20poly1305` (XChaCha20-Poly1305) + `argon2` (Argon2id):** RustCrypto crates, both mature. Argon2id is the modern KDF recommendation (better GPU resistance than age's scrypt). But you own the file format: header versioning, salt/nonce layout, chunking for large files, KDF-parameter agility, and you lose CLI interop. Worth it only if Argon2id is a hard requirement or you need format control (e.g., seekable archives). **[High]** ([docs.rs/chacha20poly1305](https://docs.rs/chacha20poly1305), [RustCrypto argon2](https://docs.rs/argon2))
- Middle option: `age` with an X25519 identity file, where the identity file itself is encrypted with Argon2id — complexity without much gain for v1. **[Low–Medium]**
- **Keychain: use `keyring` crate (4.1.6), not Stronghold.** `keyring` wraps Windows Credential Manager / macOS Keychain / Secret Service (Linux) behind one API — the same trust model users already accept for browser/git credentials ([crates.io/keyring](https://crates.io/crates/keyring), [discussion tauri#7846](https://github.com/tauri-apps/tauri/discussions/7846)). Stronghold is a password-locked snapshot vault — the wrong shape here (you'd need a password to unlock the store holding the password), it's heavy (IOTA engine), and community guidance increasingly steers away from it; keep the passphrase in the real OS keychain instead. **[High for keyring; Medium on Stronghold's long-term status — its docs page is still live with no formal deprecation notice, but it needs upstream build workarounds and is widely considered a poor fit]**
  - Linux caveat: Secret Service requires a running keyring daemon (gnome-keyring/KWallet); on headless/minimal setups fall back to prompting per-session. **[Medium]**
- **Recovery implications (must be stated in UX):** passphrase-only encryption means a forgotten passphrase = permanently unreadable vault; the keychain copy is a convenience cache, not an escrow (it doesn't survive machine loss — which is exactly the disaster-recovery scenario). Mitigations: (a) force the user to re-type the passphrase at vault creation, (b) offer a printable recovery code (age: add a second recipient), (c) never silently rotate the vault key. **[High — inherent to the design]**

## 4. Session-browser index: rusqlite + FTS5

**Recommendation:** `rusqlite` 0.40.2 with `features = ["bundled"]` and FTS5 enabled. **[High]**

- `bundled` compiles SQLite in — zero system dependencies on all three OSes, which serves "simplicity of install" directly; FTS5 is available via the `fts5` feature flag on rusqlite's bundled build for future full-text search over session content. **[High]** ([crates.io/rusqlite](https://crates.io/crates/rusqlite))
- **Why not sqlx (0.9.0):** sqlx's headline features — async I/O and compile-time query checking against a live DB — pay off for network databases under concurrency. For a local single-process SQLite index, async adds pool/runtime ceremony for no latency win (SQLite is sync at the bottom anyway), the SQLite driver is sqlx's least-mature backend, and compile-time checks complicate CI (needs a prepared DB or offline mode). rusqlite is sync, thin, and battle-tested (92M+ downloads). Run DB work on a dedicated thread / `tauri::async_runtime::spawn_blocking` from commands. **[High for the tradeoff; Medium on "least-mature backend" characterization]**
- Design note: the index is derived data — rebuildable by rescanning the vault. That makes schema migrations trivial (drop and rebuild) and means index corruption is never data loss. Keep the index *outside* the synced vault folder (per-machine, in app-data dir) to avoid SQLite-over-cloud-sync corruption, which is a classic failure mode (sync engines racing WAL/db files). **[High — well-known SQLite guidance]**
- Middle option: `tauri-plugin-sql` exposes sqlx to the frontend; skip it — Backpack's queries belong in Rust where the indexer lives, and plugin-sql weakens the boundary (SQL from webview). **[Medium]**

## 5. Updater + installers + signing costs

**Recommendation:** `tauri-plugin-updater` with a static `latest.json` on GitHub Releases; NSIS installer on Windows, notarized .dmg on macOS, AppImage (+ .deb as unsupported-for-updates extra) on Linux. **[High]**

- **Updater mechanics** ([v2.tauri.app/plugin/updater](https://v2.tauri.app/plugin/updater/)): updates are minisign-signed with a keypair you generate (`tauri signer generate`); the public key ships in `tauri.conf.json` and signature verification cannot be disabled. Supported update artifacts: Windows NSIS `.exe` / MSI; macOS `.app.tar.gz`; Linux AppImage. **`.deb`/`.rpm` installs are NOT auto-updatable** — ship AppImage as the primary Linux artifact if you want updates, offer .deb for people who prefer apt-managed (they update manually). **Guard the updater private key like production credentials: if lost, existing installs can never update again.** **[High — official docs]**
- **Windows:** NSIS over MSI — NSIS supports per-user install (no UAC), better updater UX, and both are first-class in Tauri's bundler. Signing options ([v2.tauri.app/distribute/sign/windows](https://v2.tauri.app/distribute/sign/windows/)):
  - **Azure Trusted Signing (being rebranded "Artifact Signing"): $9.99/month Basic (5,000 signatures/mo), open to individual developers, integrates with Tauri config.** This is the modern budget path. ([Azure pricing](https://azure.microsoft.com/en-gb/pricing/details/trusted-signing/), [MS announcement](https://techcommunity.microsoft.com/blog/microsoft-security-blog/trusted-signing-is-now-open-for-individual-developers-to-sign-up-in-public-previ/4273554)) **[High]** Caveat: identity validation required; individual sign-up has been in public preview with some regional/eligibility friction. **[Medium]**
  - OV certificate: roughly $200–400/yr; since June 2023 requires HSM/token storage, so cloud-HSM resellers (SSL.com eSigner, Certum, etc.) are the practical route. SmartScreen reputation builds slowly — early downloads still warn. **[Medium]**
  - EV certificate: ~$400+/yr, hardware token or cloud HSM, immediate SmartScreen reputation. ([Tauri discussion #5739](https://github.com/tauri-apps/tauri/discussions/5739)) **[Medium–High]**
  - **Unsigned:** SmartScreen "Windows protected your PC" full-screen warning on every fresh download; users must click "More info → Run anyway". Fatal for casual users, tolerable for a dev-tool audience in early beta. **[High]**
- **macOS:** Apple Developer Program $99/yr; notarization is free with membership and Tauri automates it (`APPLE_API_KEY` env vars in CI). Distribute a notarized .dmg. **Unsigned/un-notarized:** Gatekeeper blocks with "cannot be opened because the developer cannot be verified"; on macOS Sequoia+ the right-click-Open bypass was removed — users must dig into System Settings → Privacy & Security. Effectively mandatory to pay the $99 for public distribution. ([Apple support](https://support.apple.com/en-us/102445), [notarization guide](https://www.swiftyn.com/learn/macos/macos-notarization-guide-gatekeeper)) **[High for costs; Medium for exact Sequoia bypass details]**
- **Linux:** no code-signing gate; AppImage just runs (user marks executable). Optional: sign .deb repo with GPG later. **[High]**
- **Realistic annual cost to ship signed on all three: ~$219/yr** ($99 Apple + $120 Azure Trusted Signing) — worth budgeting from day one because switching signing identities later resets SmartScreen reputation. **[Medium]**
- Alternative: CrabNebula Cloud (from the Tauri maintainers' company) hosts update artifacts/CDN with a free OSS tier; nice but adds a vendor — static JSON on GitHub Releases is zero-cost and sufficient. **[Medium]**

## 6. Frontend + IPC

**Recommendation:** **Svelte 5 + TypeScript + Tailwind CSS v4, themed via CSS custom properties; Vite; Tauri `Channel` API for progress streaming.** **[Medium–High — partly judgment]**

- **Framework:** All majors are first-class in `create-tauri-app`. Svelte 5 (runes) gives the smallest bundles and fastest webview startup — visible as "crisp" in a desktop shell — with simpler day-to-day code than React for an app this size (a browser/list/detail UI + settings + progress toasts). SolidJS is the close runner-up (best raw benchmarks, smaller community); React is the fallback if you want shadcn/ui and the largest component ecosystem — at the cost of the heaviest runtime. ([CrabNebula on UI libs for Tauri](https://crabnebula.dev/blog/the-best-ui-libraries-for-cross-platform-apps-with-tauri/), [framework comparison](https://www.frontendtools.tech/blog/best-frontend-frameworks-2025-comparison)) Honest note: this is a taste call — for a solo/small project, pick the one you're fastest in; none of them is a wrong answer inside Tauri. **[Medium]**
- **Theming:** define semantic design tokens as CSS custom properties on `:root` / `[data-theme="dark"]`; Tailwind v4 is CSS-first and consumes CSS variables natively (`@theme`), so light/dark/custom themes are a single attribute swap with no JS re-render. Respect `prefers-color-scheme` as the default, persist override in settings. Component base: Skeleton UI or shadcn-svelte for polished primitives, or headless (Melt UI/Bits UI) + custom tokens for a fully bespoke sleek look. **[High for mechanism; Medium for library picks]**
- **Window polish for "sleek":** custom titlebar via `decorations: false` + `data-tauri-drag-region` is doable but costs per-OS edge cases (snap layouts on Windows, traffic lights on macOS); recommend native decorations in v1, revisit later. **[Medium]**
- **IPC patterns** ([v2.tauri.app/develop/calling-frontend](https://v2.tauri.app/develop/calling-frontend/), [IPC concepts](https://v2.tauri.app/concept/inter-process-communication/)):
  - Commands (`#[tauri::command]`, async) for request/response: list sessions, search, trigger backup/restore.
  - **`Channel<T>` for streaming progress** — pass a channel into a command (`on_progress: Channel<ProgressEvent>`); Tauri docs explicitly recommend channels over events for ordered, high-frequency streams (download progress is their own example). Use it for backup/restore/import progress bars.
  - Global events (`app.emit`) only for low-frequency broadcasts: "new session detected", "vault locked/unlocked", tray-triggered navigation.
  - Keep payloads small (JSON-serialized IPC is slow for MB-scale data — ~200 ms for 3 MB reported in [tauri discussion #11915](https://github.com/orgs/tauri-apps/discussions/11915)); never ship session file bodies wholesale to the webview — paginate/preview from Rust. **[High]**

---

## Sources (primary)

- notify / debouncer: https://docs.rs/notify/latest/notify/ · https://docs.rs/notify-debouncer-full/latest/notify_debouncer_full/ · https://crates.io/crates/notify
- Tauri tray/autostart: https://v2.tauri.app/learn/system-tray/ · https://v2.tauri.app/plugin/autostart/ · https://github.com/tauri-apps/tauri/issues/8982
- age / crypto: https://docs.rs/age · https://age-encryption.org/v1 · https://docs.rs/chacha20poly1305 · https://crates.io/crates/keyring · https://github.com/tauri-apps/tauri/discussions/7846
- SQLite: https://crates.io/crates/rusqlite · https://crates.io/crates/sqlx
- Updater/signing: https://v2.tauri.app/plugin/updater/ · https://v2.tauri.app/distribute/sign/windows/ · https://azure.microsoft.com/en-gb/pricing/details/trusted-signing/ · https://techcommunity.microsoft.com/blog/microsoft-security-blog/trusted-signing-is-now-open-for-individual-developers-to-sign-up-in-public-previ/4273554 · https://github.com/tauri-apps/tauri/discussions/5739 · https://support.apple.com/en-us/102445
- Frontend/IPC: https://v2.tauri.app/develop/calling-frontend/ · https://v2.tauri.app/concept/inter-process-communication/ · https://crabnebula.dev/blog/the-best-ui-libraries-for-cross-platform-apps-with-tauri/ · https://github.com/orgs/tauri-apps/discussions/11915
