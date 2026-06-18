# Daily webxdc game — template

[`daily-template.html`](daily-template.html) is a reusable scaffold for building any
**"one puzzle per day, shared in the chat"** webxdc game (à la Wordle). You write a small
game adapter; the template handles all the daily-game plumbing.

It's a single self-contained file — it even inlines a tiny dev webxdc stub, so you can open
it directly in a browser to develop, and a real webxdc host (Delta Chat, etc.) transparently
replaces the stub at runtime.

## What you get for free

- **Per-instance random seed** — each copy of the app is its own series of puzzles.
- **First-opener timezone setup** — the first person picks the daily reset timezone; it's
  broadcast and adopted by everyone (lowest-serial config wins), so the whole group shares
  one schedule.
- **Timezone-aware day numbering** + a live countdown to the next reset (DST-safe via `Intl`).
- **Date navigation** — browse back through past days; they render **read-only**. You can't
  go past today or before the instance's first day.
- **Daily leaderboard** (medals, you-highlight) + **Overall standings** (won/played, average,
  🏆 days-topped, 🔥 streak, points), opened full-screen.
- **Auto-sharing to the chat**: a silent `info` line on every play, and a `notify` ping only
  when someone overtakes the day's #1 (never for the lone first player).
- **Local persistence + replay** — your results survive reloads; on boot the app replays all
  updates to rebuild the shared state.
- **Multi-device aware** — results are keyed by your account address, so the leaderboards
  match on every device. If you play on one device, the others **adopt that synced result**:
  they show the day as played, render it read-only, and won't offer a duplicate attempt.
- **Live presence** — a "who's here now" indicator via the webxdc realtime channel
  (`joinRealtimeChannel`). Each client pings every few seconds; peers unseen for ~12s drop
  off. (The dev stub has no peers, so it's hidden until real players join.)

## Quick start

1. Copy `daily-template.html`.
2. Edit the **`GAME`** object at the top of the `<script>` (and swap the `.demo` CSS for your
   own). That's the only part you touch.
3. Open the file in a browser to develop. Package it as a `.xdc` to ship (see *Packaging*).

The template ships with a working **"guess the number"** demo so it runs as-is.

## The `GAME` adapter

Everything game-specific lives in one object. Required fields are in **bold**.

| Field | Type | Purpose |
|-------|------|---------|
| **`name`** | string | App name — title bar and headers. |
| **`makePuzzle(rng, day)`** | fn → puzzle | Build the day's puzzle deterministically. `rng()` is a seeded PRNG (0–1); `day` is the absolute day index. Return any object your `render` understands. |
| **`render(root, puzzle, ctx)`** | fn | Draw the puzzle into `root` (already emptied). Call `ctx.submit(result)` **once** when the official attempt finishes. See `ctx` below. |
| **`points(result)`** | fn → number | Points awarded toward the Overall standings for a recorded result. |
| `verb` | string | Chat phrasing: `"<name> <verb> #5 — <label>"`. Default `"played"`. |
| `helpHTML` | string | HTML shown in the "How to play" modal. |
| `label(result)` | fn → string | Short text for the leaderboard cell. Defaults to `result.display`. |
| `formatAvg(mean)` | fn → string | Formats the Overall **Avg** column. Defaults to `mean.toFixed(1)`. |
| `scoreHelp` | string | One-line explainer under the Overall table. |

### The result contract

Your game reports each finished attempt by calling `ctx.submit(result)`, where `result` is:

```js
{
  score:   42,          // number — HIGHER IS BETTER. Used for daily ranking + Overall avg.
  win:     true,        // boolean — counted in "Won" and used for streaks.
  display: "3 tries"    // string  — shown on the leaderboard / read-only view.
}
```

If your game is "lower is better" (golf, guesses, time), invert it into `score` — e.g.
`score = MAX_TRIES + 1 - tries`. Keep the human-friendly form in `display`.

### The `ctx` passed to `render`

| `ctx` field | Meaning |
|-------------|---------|
| `ctx.playable` | `true` only for **today**, not yet played. Otherwise render a read-only view. |
| `ctx.previous` | Your saved `result` for this day, or `null`. |
| `ctx.isToday` | Whether the viewed day is today. |
| `ctx.day` | The absolute day index being viewed. |
| `ctx.submit(result)` | Record + broadcast the official attempt (today only, once). |
| `ctx.toast(msg)` | Show a transient message. |

After `ctx.submit`, the framework re-renders the day read-only (with `ctx.previous` set) and
refreshes the boards — your `render` should handle the `!ctx.playable` case.

## Example mapping (Even Split)

The sibling cutting game maps onto the adapter like this:

```js
var GAME = {
  name: "Even Split",
  verb: "cut",
  makePuzzle: function (rng) { return buildShape(rng); },      // a geometric shape
  render: function (root, shape, ctx) {
    // draw the shape on a canvas; on drag-release:
    var minority = areaRatioOfCut(...);                        // 0..50, higher = closer to 50:50
    ctx.submit({ score: minority, win: minority >= 48, display: minority.toFixed(1) });
  },
  points: function (r) { return r.win ? 3 : r.score >= 45 ? 2 : 1; },
};
```

## How the shared state works

Two webxdc update types flow over `sendUpdate` / `setUpdateListener`:

- **`config`** — `{ type:"config", seed, tz, startDay }`. Sent by the first opener; everyone
  adopts the earliest one (lowest serial). Establishes the puzzle series and reset schedule.
- **`result`** — `{ type:"result", day, addr, name, score, win, display }`. One per player per
  day. Carries chat metadata:
  - `info` — a silent info message, e.g. `"Alice cut #5 — 49.8"` (added every play).
  - `notify: { "*": ... }` — only when the result **overtakes** the day's leader.
  - `summary` — current daily leader + overall winner, shown on the app's chat bubble.
  - `document` — `"#<puzzle number>"`.

Local storage keys: `daily-template-mine-v1` (your own results) and, in dev only,
`__daily_template_sim__` (the stub's simulated update log). Rename these per app.

## Packaging as a `.xdc`

A `.xdc` is just a zip containing at least `index.html` + `manifest.toml`:

```
your-game/
  index.html      ← your filled-in copy of daily-template.html
  manifest.toml   ← name = "Your Game"
  icon.png        ← optional app icon
```

Zip those into `your-game.xdc` and send it into a chat. (The inlined dev stub is harmless in
production — the host injects its own `webxdc.js`.)

## Tuning ideas

- **Lower-is-better Avg display** — invert in `score`, pretty-print in `formatAvg`.
- **Loss penalties / decay** in `points` — tune to taste.

## Credits

The structure follows the daily-webxdc pattern established by
[webxdc-word-challenge](https://github.com/trosel/webxdc-word-challenge) (`daily.html`).
