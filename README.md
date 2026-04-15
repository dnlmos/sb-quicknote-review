# sb-quicknote-review

A [SilverBullet](https://silverbullet.md) **v2** Space Lua plugin for quickly reviewing, triaging, and deleting quick notes stored in an `Inbox/` folder.

> Written by [Claude Code](https://claude.com/claude-code) (Anthropic's CLI for Claude) on behalf of the repo owner. The owner reviewed and published it but did not hand-write the code.

## What it does

When you dump lots of one-line thoughts into an `Inbox/` folder (e.g. via SilverBullet's "Quick Note" command), this plugin gives you a simple review loop:

1. Trigger a slash command to jump to the **oldest** quick note.
2. On every Inbox page, three action buttons appear at the top:

   **Delete quick note** · **Previous quick note** · **Next quick note**
3. Walk through the notes, extract anything worth keeping, then delete.

Notes are ordered alphabetically by page name, which is chronological if your filenames are timestamps (e.g. `Inbox/2026-04-15/13-37-36`).

## Requirements

- SilverBullet **v2** (Space Lua). This plugin will not work on v1 — it uses the v2 Lua API (`command.define`, `slashCommand.define`, `event.listen { name = "hooks:renderTopWidgets" }`, etc.).

## Installation

Space Lua code loads from any markdown page with a `space-lua` code fence. Pick one of these options:

### Option A — copy the file into your space

1. Copy [`QuickNotes.md`](QuickNotes.md) into your SilverBullet space. A conventional location is `Library/QuickNotes.md`, but anywhere under your space root will work.
2. In SilverBullet, reload the space so it picks up the new Lua. Either:
   - Run the command **System: Reload** (`Ctrl-Alt-R` / `⌃⌥R`), or
   - Refresh the browser tab.

### Option B — paste into an existing page

Open the raw [`QuickNotes.md`](QuickNotes.md), copy the `space-lua` code block (between the triple backticks), and paste it into any page in your space. Reload as above.

## Usage

| Action | How to trigger |
|---|---|
| Start review (jump to oldest Inbox note) | Slash command `/review-notes` *or* command palette → **Quick Notes: Review Oldest** |
| Delete current note, move on | **Delete quick note** button *or* command palette → **Quick Notes: Delete Current** |
| Next note | **Next quick note** button *or* command palette → **Quick Notes: Next** |
| Previous note | **Previous quick note** button *or* command palette → **Quick Notes: Previous** |

The action buttons only render when the current page starts with `Inbox/`, so they won't clutter other pages.

After deletion, the plugin navigates to the next Inbox note (or the previous, or `index` if the Inbox is empty).

## Customisation

The plugin assumes your quick notes live under `Inbox/`. If you use a different prefix, edit the two `string.startsWith(..., "Inbox/")` checks in `QuickNotes.md`.

## License

[MIT](LICENSE)

## Attribution

Code written by [Claude Code](https://claude.com/claude-code) on model Claude Opus 4.6. Commit messages include co-author attribution.
