# sb-quicknote-review

A [SilverBullet](https://silverbullet.md) **v2** Space Lua library for quickly reviewing, triaging, and deleting quick notes stored in an `Inbox/` folder.

> Written by [Claude Code](https://claude.com/claude-code) (Anthropic's CLI for Claude) on behalf of the repo owner. The owner reviewed and published it but did not hand-write the code.

## What it does

When you dump lots of one-line thoughts into an `Inbox/` folder (e.g. via SilverBullet's "Quick Note" command), this library gives you a simple review loop:

1. From the command palette, run **Quick Notes: Review Oldest** to jump to the oldest quick note.
2. On every page under `Inbox/`, three action buttons appear at the top:

   **Delete quick note** · **Previous quick note** · **Next quick note**
3. Walk through the notes, extract anything worth keeping, then delete.

Notes are ordered alphabetically by page name, which is chronological if your filenames are timestamps (e.g. `Inbox/2026-04-15/13-37-36`).

## Requirements

- SilverBullet **v2** (Space Lua). This will not work on v1 — it uses the v2 Lua API (`command.define`, `event.listen { name = "hooks:renderTopWidgets" }`, etc.).

## Installation

### Recommended — built-in `Library: Install`

SilverBullet v2 can install libraries directly from a URL and track them for updates.

1. Open the command palette (`Ctrl-/` or `⌘-/`) and run **Library: Install**.
2. When prompted for **URI**, paste:

   ```
   https://github.com/johnhunt/sb-quicknote-review/blob/main/PLUG.md
   ```
3. SilverBullet downloads the library, writes it to `Library/QuickNoteReview` in your space, and reloads. Done.

To update later, run **Library: Update All** (or **Update** from the Library Manager). To remove, use the Library Manager's **Remove** button.

### Alternative — copy the file manually

1. Copy [`PLUG.md`](PLUG.md) into your space (anywhere under the space root works; `Library/QuickNoteReview.md` matches the convention).
2. Run **System: Reload** (`Ctrl-Alt-R` / `⌃⌥R`) or refresh the browser tab.

## Usage

| Action | How to trigger |
|---|---|
| Start review (jump to oldest Inbox note) | Command palette → **Quick Notes: Review Oldest** |
| Delete current note, move on | **Delete quick note** button *or* command palette → **Quick Notes: Delete Current** |
| Next note | **Next quick note** button *or* command palette → **Quick Notes: Next** |
| Previous note | **Previous quick note** button *or* command palette → **Quick Notes: Previous** |

The action buttons only render when the current page starts with `Inbox/`, so they don't clutter other pages.

After deletion, the plugin navigates to the next Inbox note (or the previous, or `index` if the Inbox is empty).

## Customisation

The library assumes your quick notes live under `Inbox/`. If you use a different prefix, edit the two `string.startsWith(..., "Inbox/")` checks in `PLUG.md`.

## License

[MIT](LICENSE)

## Attribution

Code written by [Claude Code](https://claude.com/claude-code) on model Claude Opus 4.6. Commit messages include co-author attribution.
