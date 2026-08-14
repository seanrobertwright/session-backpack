# Prototype — UI direction (dashboard & session browser)

Wayfinder ticket: [Prototype: UI direction — dashboard & session browser](https://github.com/seanrobertwright/session-backpack/issues/16)
Map: [Wayfinder map: Backpack v1 spec](https://github.com/seanrobertwright/session-backpack/issues/1)

## Question

What should Backpack look and feel like? Three directions for the vault dashboard
and the session browser, concrete enough to react to.

## Shape

One self-contained `index.html` — no build step, open it in any browser.

- `?variant=A|B|C` switches direction. Floating bar bottom-centre, or `←` / `→`.
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

_Not yet locked — awaiting the HITL reaction on ticket #16._

Record here: which direction won, which pieces were stolen from the losers, and
why. Then delete the losing variants and the switcher.
