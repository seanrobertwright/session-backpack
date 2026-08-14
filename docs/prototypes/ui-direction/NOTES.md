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

The refinement layered on top: the three are **views over the same two nouns**
(captures and sessions), not three apps — same data, same actions, same reader,
different navigation shape. So the switcher goes on the *browser*, and Home is
built **once**:

- **Home** is always A's health band + stats + a "Needs attention" exception list.
  One answer to "am I safe?", and the app still *opens* on backup status — which
  keeps [#8](https://github.com/seanrobertwright/session-backpack/issues/8)'s
  positioning (backup is the product, browsing is supporting cast) intact even
  though a full browser ships.
- **Sessions** carries a segmented **List / Table / Timeline** switcher.
- **One filter sidebar** is shared by all three views, and so is the selection —
  filter to `pilot` in List, switch to Timeline, still `pilot`. That shared state
  is the actual claim being tested; without it the switcher is three apps in a
  trenchcoat.

Cost of this over picking one: one dashboard instead of three, plus three list
bodies instead of one. The reader, the filters, the actions and the vault model
are all built once regardless.

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

**All three win, as user-selectable views over one home.** See "Outcome" above.

Open sub-decision at time of writing: whether Home stays fixed (as built) or is
itself one of the switchable views. Built fixed, on the argument that a dashboard
that changes shape gives "am I safe?" more than one answer.

When folding into real code: keep U, delete A, B and C along with the prototype
switcher bar. The variant components were written under prototype constraints —
rewrite, don't promote.
