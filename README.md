# sb-quicknote-review

A [SilverBullet](https://silverbullet.md) **v2 / edge** Space Lua library for reviewing, triaging, snoozing and deleting quick notes stored in an `Inbox/` folder.

> Written by [Claude Code](https://claude.com/claude-code) (Anthropic's CLI for Claude) on behalf of the repo owner. The owner reviewed and published it but did not hand-write the code.

## What it does

When you dump lots of one-line thoughts into an `Inbox/` folder (via SilverBullet's **Quick Note** command, `Ctrl-q q`), this library gives you a review loop:

1. Run **Quick Notes: Review** (`Ctrl-q r`) to jump to the oldest note waiting for you.
2. On every page under `Inbox/`, a review bar appears above the note:

   **Snooze** · **Snooze for…** · **Delete** · **‹ Prev** · **Next ›**

   with a status line — `Note 3 of 17 due · 4 snoozed · snoozed 2×, back in 3 days · snooze → in 1 day`.
3. Act on the note and delete it, or snooze it and move on.

Notes are ordered by page name, which is chronological if your filenames are timestamps (e.g. `Inbox/2026-04-15/13-37-36`).

## Snoozing: spaced repetition for your inbox

**Snooze** writes a `snoozeUntil` stamp into the note's frontmatter and drops the note out of the review queue until that time passes:

```yaml
---
snoozeUntil: "2026-11-28 09:30"
snoozeCount: 2
---
```

Snoozing follows classic spaced repetition: each snooze of the same note pushes it further out than the last one.

| Snooze | Back in |
|---|---|
| 1st | 1 day |
| 2nd | 3 days |
| 3rd | 7 days |
| 4th | 21 days |
| 5th and beyond | 60 days |

The idea: if you've reviewed a note a few times without deleting it, it's low-urgency and can fade into the background rather than demanding attention every day. If you'd rather have the opposite — a note that nags you *more* the more you dodge it (7d → 3d → 1d → 12h → 4h) — that's one config line, see [Configuration](#configuration).

State lives in the note's own frontmatter, not in a local key-value store, so snoozes sync with your space and follow you across devices.

## Requirements

- SilverBullet **v2 / edge**. This uses edge APIs: `view.define` with `dock = "page-top"` (the old `hooks:renderTopWidgets` is deprecated), `index.patchFrontmatter`, `index.extractFrontmatter`, `editor.filterBox`, and string refs for `editor.navigate`.

## Installation

### Recommended — built-in `Library: Install`

1. Open the command palette (`Ctrl-/` or `⌘-/`) and run **Library: Install**.
2. When prompted for **URI**, paste:

   ```
   https://github.com/dnlmos/sb-quicknote-review/blob/main/PLUG.md
   ```
3. SilverBullet downloads the library to `Library/QuickNoteReview` and reloads.

To update later, run **Library: Update All**. To remove, use the Library Manager's **Remove** button.

### Alternative — copy the file manually

1. Copy [`PLUG.md`](PLUG.md) into your space as `Library/QuickNoteReview.md`.
2. Run **System: Reload** (`Ctrl-Alt-R` / `⌃⌥R`).

## Usage

| Action | Button | Command | Key |
|---|---|---|---|
| Jump to the oldest note due | — | **Quick Notes: Review** | `Ctrl-q r` |
| Snooze (spaced repetition) | **Snooze** | **Quick Notes: Snooze** | `Ctrl-q s` |
| Snooze for a delay you pick | **Snooze for…** | **Quick Notes: Snooze For** | `Ctrl-q f` |
| Clear a snooze | — | **Quick Notes: Unsnooze** | `Ctrl-q u` |
| Delete the note, move on | **Delete** | **Quick Notes: Delete Current** | `Ctrl-q x` |
| Next note due | **Next ›** | **Quick Notes: Next** | `Ctrl-q n` |
| Previous note due | **‹ Prev** | **Quick Notes: Previous** | `Ctrl-q p` |
| Browse everything, due and snoozed | — | **Quick Notes: Queue** | `Ctrl-q l` |

**Quick Notes: Queue** opens a picker listing every note due now, then the snoozed ones with when each comes back. It docks to a sidebar if you'd rather keep it open.

The review bar only renders on pages under `Inbox/`, so it doesn't clutter the rest of your space. Being a view, it can be moved to the bottom of the page or closed from its hover menu.

**Prev** / **Next** walk the *due* queue, skipping snoozed notes. If you land on a snoozed note anyway (from the queue picker, say), they fall back to walking every note so you can browse your way out.

After a delete or snooze, the plugin moves to the next note due, or the previous one if that was the last.

## Configuration

Both keys are optional. Put them in your `Library/Config` page:

```lua
-- Where quick notes live (trailing slash optional)
config.set("quickNotes.folder", "Inbox/")

-- Delays, in seconds, for the 1st, 2nd, 3rd… snooze of the same note.
-- The last entry repeats forever.
-- For a "nag" curve instead -- each snooze brings the note back sooner,
-- until you deal with it:
config.set("quickNotes.snoozeLadder", {
  7 * 24 * 3600, 3 * 24 * 3600, 1 * 24 * 3600, 12 * 3600, 4 * 3600,
})
```

To drop a key binding you'd rather keep for something else:

```lua
command.update { name = "Quick Notes: Snooze", key = "", mac = "" }
```

## License

[MIT](LICENSE)

## Attribution

Code written by [Claude Code](https://claude.com/claude-code). Commit messages include co-author attribution.
