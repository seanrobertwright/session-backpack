# Prototype — UI direction (dashboard & session browser)

Wayfinder ticket: [Prototype: UI direction — dashboard & session browser](https://github.com/seanrobertwright/session-backpack/issues/16)
Map: [Wayfinder map: Backpack v1 spec](https://github.com/seanrobertwright/session-backpack/issues/1)

## Question

What should Backpack look and feel like? Three directions for the vault dashboard
and the session browser, concrete enough to react to.

## Outcome — all three ship, as user-selectable views

Sean's call after the first round: _"I actually love all three — can we have it so
the user can select their view from one of the 3?"_ That is now **variant U**, the
default when the file opens. A, B and C are kept alongside it as the originals the
composition came from.

**Home switches too** — settled in the second round. The preference is therefore
**global and singular**: pick Console, Navigator or Timeline once, and both Home
and Sessions adopt that shape. Each original direction stays intact end to end
rather than being mixed into nine hybrids, and there is one setting to manage
rather than two.

| Shape | Home | Sessions |
|---|---|---|
| **Console** | Shield band + stat strip + "Needs attention" | Dense table, inline session detail |
| **Navigator** | Vault pane — path, encryption, stat tiles, exceptions, machines | List + reading pane |
| **Timeline** | Health ribbon + machine lanes + recent capture stream | Capture spine + scrubber, session opens over the top |

What the **shell** owns, and therefore what never changes when the shape does:

- the rail (Home / Sessions / Machines / Settings),
- the filter facets and the current filter,
- the selected session.

Filter to `pilot` in Timeline, switch to Console, still `pilot`. That shared state
is the whole claim: without it the switcher is three apps in a trenchcoat.

**On positioning.** With Home switchable, the app no longer always *opens* on a
backup-status band — Navigator's Home leads with the vault, Timeline's with a
capture stream. All three still answer "am I safe?" above the fold (band, stat
grid, or ribbon respectively), but this is a real softening of
[#8](https://github.com/seanrobertwright/session-backpack/issues/8)'s "browsing is
supporting cast". Recorded deliberately, not by accident.

Cost: three Home treatments and three browser treatments. The reader, filters,
actions, adapters and vault model are built once regardless.

## Theming — decided

- **Auto is the shipping default**: follow the OS, Slate in dark, Paper in light,
  nothing to configure. It is now an explicit fourth option in the picker rather
  than a hidden "click the active skin to turn it off" state.
- **Terminal ships as a real user-facing option**, not just a proof that the token
  seam holds. It swaps the typeface and drops the corner radius to `0` as well as
  the palette — so any future skin may change more than hue.

## Shape

One self-contained `index.html` — no build step, open it in any browser.

- `?variant=U|A|B|C` switches direction (U is the default). Floating bar
  bottom-centre, or `←` / `→`.
- `?skin=slate|paper|terminal` switches theme, from the title bar. With no skin
  chosen the page follows the viewer's light/dark setting.
- Data is fake but **identical across all three variants** — same 3 machines,
  9 adapters, 14 sessions — so the comparison is like-for-like and not a
  content contest.

## The three directions

| | Direction | Information hierarchy | Primary affordance | Bet it makes |
|---|---|---|---|---|
| **A** | **Console** | Health band → adapter table → sessions as a tab | "Am I safe?" answered on sight, then *Back up now* | Backpack is a **safety utility**. You open it when something feels wrong, confirm you're covered, and close it. Sessions are inventory, not content. |
| **B** | **Navigator** | Source tree → session list → reading pane | Navigate and read | Backpack is an **archive you live in**. The dashboard is one item in the sidebar; the transcript is the product. |
| **C** | **Timeline** | One chronological spine of captures, machines as coloured lanes | Scrub back through time | Backpack is **Time Machine, literally**. Time is the only index; sessions surface out of the capture that caught them. |

They disagree on purpose. A is a table, B is a tree-list-detail, C has no
navigation chrome at all — picking one is picking what Backpack *is*.

## Theming

Every colour is a CSS custom property on `:root`; a skin is nothing but a
re-declaration of that token set behind a `[data-skin]` attribute. The `terminal`
skin also swaps `--ui` to the mono stack and `--r` to `0px`, to prove the seam
holds when a theme changes more than hue. This mirrors the real plan from
[Research: Tauri v2 stack](https://github.com/seanrobertwright/session-backpack/issues/9)
(Tailwind v4 CSS variables + `data-theme` swap).

Semantic colour (backed up / attention / failed) is deliberately a **separate
axis** from the accent, so status never competes with branding. The chrome is
cold; warmth means something.

## Decisions the prototype is quietly asserting

Worth agreeing or rejecting explicitly, since each shapes the spec:

1. **The header answers "am I safe?" before anything else** (all three variants
   lead with vault health, not with content).
2. **Restore is a per-session action**, sitting next to Export in the session
   header — not a separate wizard.
3. **The unlock state is ambient**, shown as a quiet "Unlocked · remembered on
   this machine" line rather than a modal — matching the lazy-unlock decision in
   [Decide: encryption & key management UX](https://github.com/seanrobertwright/session-backpack/issues/14).
4. **Machines are a first-class dimension** (colour-coded everywhere), not a
   filter buried in settings.
5. **Adapter health is a list of exceptions**, not a wall of green ticks.

## Verdict

**Locked.** All three ship as one global, user-selectable view shape covering both
Home and Sessions; Auto/OS-following theme by default with Terminal as a real
option. Nothing about this prototype is still open.

When folding into real code: keep U, delete A, B and C along with the prototype
switcher bar. The variant components were written under prototype constraints —
no tests, no error handling, `innerHTML` throughout with hardcoded fixtures —
so rewrite, don't promote.

Two things this prototype does **not** answer, left for later tickets:

- What the session reader actually renders per adapter format (markdown, code,
  tool calls, thinking blocks, images) — it fakes a generic six-turn transcript.
- Where the view-shape and theme preferences are stored. They are per-machine UI
  state, so almost certainly alongside the per-machine SQLite index rather than
  in the synced vault — but that is the vault-format ticket's call, not this one's.
