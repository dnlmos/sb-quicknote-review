---
name: "Library/QuickNoteReview"
tags: meta/library
---

# Quick Notes Review

A [SilverBullet](https://silverbullet.md) **v2 / edge** Space Lua library for reviewing and triaging quick notes stored under `Inbox/`.

Open any page under `Inbox/` and a review bar appears above the note:

**Snooze** · **Snooze for…** · **Delete** · **‹ Prev** · **Next ›**

Run **Quick Notes: Review** (`Ctrl-q r`) to jump to the oldest note that's waiting for you, then walk the queue.

## Snoozing

**Snooze** is for notes you can't deal with right now. It writes a `snoozeUntil` stamp into the note's frontmatter and drops it out of the review queue until that time passes.

Snoozing is deliberately *not* classic spaced repetition. Each snooze of the same note brings it back **sooner**, not later:

| Snooze | Back in |
|---|---|
| 1st | 7 days |
| 2nd | 3 days |
| 3rd | 1 day |
| 4th | 12 hours |
| 5th and beyond | 4 hours |

A note you keep dodging nags you harder until you either act on it or delete it. Override the curve with `config.set("quickNotes.snoozeLadder", {…})` — see [[#Configuration]].

## Configuration

Put these in your `Library/Config` page. Both are optional.

```lua
-- Where quick notes live (trailing slash optional)
config.set("quickNotes.folder", "Inbox/")

-- Delays, in seconds, for the 1st, 2nd, 3rd… snooze of the same note.
-- The last entry repeats forever. For classic (expanding) spaced repetition:
config.set("quickNotes.snoozeLadder", {
  1 * 24 * 3600, 3 * 24 * 3600, 7 * 24 * 3600, 21 * 24 * 3600, 60 * 24 * 3600,
})
```

To drop a key binding you don't want:

```lua
command.update { name = "Quick Notes: Snooze", key = "", mac = "" }
```

## Implementation

```space-lua
-- priority: 10

quickNotes = quickNotes or {}

-- The "nag" curve: each snooze of the same note brings it back sooner, so a
-- note you keep dodging ends up in your face. Overridable per space through
-- config.set("quickNotes.snoozeLadder", {...}).
local DEFAULT_LADDER = {
  7 * 24 * 3600, -- 1st snooze: a week
  3 * 24 * 3600, -- 2nd: three days
  1 * 24 * 3600, -- 3rd: a day
  12 * 3600,     -- 4th: twelve hours
  4 * 3600,      -- 5th and beyond: four hours
}

-- Written quoted into frontmatter, so YAML keeps it a string, and it sorts
-- lexicographically in the same order as chronologically.
local DUE_FORMAT = "%Y-%m-%d %H:%M"

-- Configuration -------------------------------------------------------------

--- The folder quick notes live in, always with a trailing slash.
function quickNotes.folder()
  local folder = config.get("quickNotes.folder", "Inbox/")
  if type(folder) ~= "string" or folder == "" then
    folder = "Inbox/"
  end
  if not string.endsWith(folder, "/") then
    folder = folder .. "/"
  end
  return folder
end

--- The snooze delays, in seconds, for the 1st, 2nd, 3rd… snooze of a note.
function quickNotes.ladder()
  local ladder = config.get("quickNotes.snoozeLadder", DEFAULT_LADDER)
  if type(ladder) ~= "table" or #ladder == 0 then
    return DEFAULT_LADDER
  end
  return ladder
end

--- Is this page one of our quick notes?
function quickNotes.isNote(page)
  return type(page) == "string" and string.startsWith(page, quickNotes.folder())
end

-- Time ----------------------------------------------------------------------

--- Formats an epoch second as the stamp we store in frontmatter.
function quickNotes.formatDue(timestamp)
  return os.date(DUE_FORMAT, timestamp)
end

--- Parses a "YYYY-MM-DD HH:MM" stamp (or a bare date) back into epoch seconds.
function quickNotes.parseDue(value)
  if value == nil then
    return nil
  end
  local s = tostring(value)
  local year, month, day, hour, minute =
    string.match(s, "^(%d%d%d%d)-(%d%d)-(%d%d)[T ](%d%d):(%d%d)")
  if not year then
    year, month, day = string.match(s, "^(%d%d%d%d)-(%d%d)-(%d%d)")
    hour, minute = "00", "00"
  end
  if not year then
    return nil
  end
  return os.time {
    year = tonumber(year),
    month = tonumber(month),
    day = tonumber(day),
    hour = tonumber(hour),
    min = tonumber(minute),
  }
end

--- Renders a number of seconds as "in 3 days" or "4 hours ago".
function quickNotes.humanDelta(seconds)
  local past = seconds < 0
  local n = math.abs(seconds)
  local value, unit
  if n < 90 * 60 then
    value, unit = math.floor(n / 60 + 0.5), "minute"
  elseif n < 24 * 3600 then
    value, unit = math.floor(n / 3600 + 0.5), "hour"
  else
    value, unit = math.floor(n / 86400 + 0.5), "day"
  end
  local label = value .. " " .. unit
  if value ~= 1 then
    label = label .. "s"
  end
  if past then
    return label .. " ago"
  end
  return "in " .. label
end

--- Seconds from now until `hour` o'clock tomorrow (at least a minute).
local function untilTomorrowAt(hour)
  local t = os.date("*t", os.time() + 86400)
  local target = os.time {
    year = t.year, month = t.month, day = t.day, hour = hour, min = 0, sec = 0,
  }
  local delta = target - os.time()
  if delta < 60 then
    delta = 60
  end
  return delta
end

-- Note state ----------------------------------------------------------------

--- The frontmatter of `page`, read from the editor when that page is the one
--- you're on (so unsaved edits count) and from the space otherwise.
function quickNotes.frontmatter(page)
  local text
  if editor.getCurrentPage() == page then
    text = editor.getText()
  else
    text = space.readPage(page)
  end
  return (index.extractFrontmatter(text)).frontmatter or {}
end

--- When this note is next due, in epoch seconds, or nil when it isn't snoozed.
--- Takes anything carrying the frontmatter keys: an indexed page object or a
--- frontmatter table.
function quickNotes.dueAt(note)
  return quickNotes.parseDue(note.snoozeUntil)
end

--- How many times this note has been snoozed.
function quickNotes.snoozeCount(note)
  return tonumber(note.snoozeCount) or 0
end

--- The delay the `count`-th snooze of a note earns. The last ladder entry
--- repeats forever.
function quickNotes.nextDelay(count)
  local ladder = quickNotes.ladder()
  return ladder[math.min(math.max(count, 1), #ladder)]
end

-- The queue -----------------------------------------------------------------

--- Every quick note, oldest first, as indexed page objects (so frontmatter
--- keys such as `snoozeUntil` are available as attributes).
function quickNotes.all()
  local prefix = quickNotes.folder()
  local result = {}
  for _, page in ipairs(query[[from p = index.pages() order by p.name]]) do
    if string.startsWith(page.name, prefix) then
      table.insert(result, page)
    end
  end
  return result
end

--- Splits the notes into the ones waiting for you (oldest first) and the ones
--- still snoozed (soonest back first).
function quickNotes.partition(now)
  now = now or os.time()
  local due, snoozed = {}, {}
  for _, note in ipairs(quickNotes.all()) do
    local at = quickNotes.dueAt(note)
    if at and at > now then
      table.insert(snoozed, note)
    else
      table.insert(due, note)
    end
  end
  table.sort(snoozed, function(a, b)
    return (quickNotes.dueAt(a) or 0) < (quickNotes.dueAt(b) or 0)
  end)
  return due, snoozed
end

--- The review queue: notes that are due now, oldest first.
function quickNotes.queue()
  local due = quickNotes.partition()
  return due
end

--- 1-based position of `page` in a list of notes, or nil.
function quickNotes.indexOf(notes, page)
  for i, note in ipairs(notes) do
    if note.name == page then
      return i
    end
  end
  return nil
end

--- The note to land on once `page` leaves the queue: the one after it, else
--- the one before it. Call this *before* deleting or snoozing, while `page` is
--- still in the queue.
local function neighbourOf(page)
  local queue = quickNotes.queue()
  local idx = quickNotes.indexOf(queue, page)
  if not idx then
    return nil
  end
  local target = queue[idx + 1] or queue[idx - 1]
  return target and target.name or nil
end

--- Flashes an error and returns false when the current page isn't a quick note.
local function requireNote(page)
  if quickNotes.isNote(page) then
    return true
  end
  editor.flashNotification(
    "Not a quick note (page isn't under " .. quickNotes.folder() .. ").",
    "error")
  return false
end

-- Navigation ----------------------------------------------------------------

--- Jumps to the oldest note that's due.
function quickNotes.review()
  local due, snoozed = quickNotes.partition()
  if #due > 0 then
    editor.navigate(due[1].name)
    return
  end
  if #snoozed > 0 then
    local at = quickNotes.dueAt(snoozed[1])
    editor.flashNotification(
      "Inbox clear. " .. #snoozed .. " snoozed, next one back " ..
      quickNotes.humanDelta(at - os.time()) .. ".", "info")
    return
  end
  editor.flashNotification("No quick notes to review!", "info")
end

--- Moves `delta` steps through the review queue from the current page.
function quickNotes.step(delta)
  local page = editor.getCurrentPage()
  local queue = quickNotes.queue()
  local idx = quickNotes.indexOf(queue, page)
  if not idx then
    -- Not in the queue: either snoozed, or not a quick note at all. Walk every
    -- note instead, so you can still browse your way out of a snoozed one.
    local all = quickNotes.all()
    local allIdx = quickNotes.indexOf(all, page)
    local target = allIdx and all[allIdx + delta]
    if target then
      editor.navigate(target.name)
    else
      quickNotes.review()
    end
    return
  end
  local target = queue[idx + delta]
  if not target then
    editor.flashNotification(
      delta > 0 and "That was the last quick note due."
        or "Already at the first quick note due.",
      "info")
    return
  end
  editor.navigate(target.name)
end

function quickNotes.gotoNext()
  quickNotes.step(1)
end

function quickNotes.gotoPrev()
  quickNotes.step(-1)
end

-- Actions -------------------------------------------------------------------

--- Applies frontmatter patches to `page`, through the editor when that page is
--- open so unsaved edits survive.
function quickNotes.patch(page, patches)
  if editor.getCurrentPage() == page then
    editor.setText(index.patchFrontmatter(editor.getText(), patches))
    editor.save()
  else
    space.writePage(page, index.patchFrontmatter(space.readPage(page), patches))
  end
end

--- Deletes the current note and moves on to the next one due.
function quickNotes.deleteCurrent()
  local page = editor.getCurrentPage()
  if not requireNote(page) then
    return
  end
  local target = neighbourOf(page)
  space.deletePage(page)
  editor.flashNotification("Deleted " .. page, "info")
  editor.navigate(target or "index")
end

--- Snoozes the current note and moves on. Without `seconds`, the delay comes
--- from the ladder: the more often this note has been snoozed, the sooner it
--- comes back.
function quickNotes.snoozeCurrent(seconds)
  local page = editor.getCurrentPage()
  if not requireNote(page) then
    return
  end
  local target = neighbourOf(page)
  local count = quickNotes.snoozeCount(quickNotes.frontmatter(page)) + 1
  local delay = seconds or quickNotes.nextDelay(count)
  local due = os.time() + delay
  quickNotes.patch(page, {
    { op = "set-key", path = "snoozeUntil", value = quickNotes.formatDue(due) },
    { op = "set-key", path = "snoozeCount", value = count },
  })
  editor.flashNotification(
    "Snoozed " .. count .. "×, back " .. quickNotes.humanDelta(delay) .. ".",
    "info")
  if target then
    editor.navigate(target)
  end
end

--- Snoozes the current note for a delay you pick.
function quickNotes.snoozeCurrentFor()
  local page = editor.getCurrentPage()
  if not requireNote(page) then
    return
  end
  local choices = {
    { name = "In 1 hour", value = "3600" },
    { name = "In 4 hours", value = "14400" },
    { name = "Tomorrow morning", value = tostring(untilTomorrowAt(9)) },
    { name = "In 3 days", value = "259200" },
    { name = "Next week", value = "604800" },
  }
  local count = quickNotes.snoozeCount(quickNotes.frontmatter(page)) + 1
  local picked = editor.filterBox("Snooze until:", choices,
    "Snooze " .. count .. " of this note. Plain Snooze would use " ..
    quickNotes.humanDelta(quickNotes.nextDelay(count)) .. ".")
  if not picked then
    return
  end
  quickNotes.snoozeCurrent(tonumber(picked.value))
end

--- Clears the snooze on the current note, putting it straight back in the
--- queue and resetting its nag counter.
function quickNotes.unsnoozeCurrent()
  local page = editor.getCurrentPage()
  if not requireNote(page) then
    return
  end
  quickNotes.patch(page, {
    { op = "delete-key", path = "snoozeUntil" },
    { op = "delete-key", path = "snoozeCount" },
  })
  editor.flashNotification("Snooze cleared, this note is due now.", "info")
end

-- Commands ------------------------------------------------------------------

command.define {
  name = "Quick Notes: Review",
  icon = "inbox",
  key = "Ctrl-q r",
  mac = "Ctrl-q r",
  run = quickNotes.review,
}

command.define {
  name = "Quick Notes: Next",
  icon = "chevron-right",
  key = "Ctrl-q n",
  mac = "Ctrl-q n",
  run = quickNotes.gotoNext,
}

command.define {
  name = "Quick Notes: Previous",
  icon = "chevron-left",
  key = "Ctrl-q p",
  mac = "Ctrl-q p",
  run = quickNotes.gotoPrev,
}

command.define {
  name = "Quick Notes: Snooze",
  icon = "alarm-clock",
  key = "Ctrl-q s",
  mac = "Ctrl-q s",
  run = function()
    quickNotes.snoozeCurrent()
  end,
}

command.define {
  name = "Quick Notes: Snooze For",
  icon = "alarm-clock-plus",
  key = "Ctrl-q f",
  mac = "Ctrl-q f",
  opensInput = true,
  run = quickNotes.snoozeCurrentFor,
}

command.define {
  name = "Quick Notes: Unsnooze",
  icon = "alarm-clock-off",
  key = "Ctrl-q u",
  mac = "Ctrl-q u",
  run = quickNotes.unsnoozeCurrent,
}

command.define {
  name = "Quick Notes: Delete Current",
  icon = "trash-2",
  key = "Ctrl-q x",
  mac = "Ctrl-q x",
  run = quickNotes.deleteCurrent,
}

-- The review bar ------------------------------------------------------------

--- A one-line summary of where you are in the queue and what snoozing costs.
function quickNotes.status(page)
  local due, snoozed = quickNotes.partition()
  local parts = {}
  local idx = quickNotes.indexOf(due, page)
  if idx then
    table.insert(parts, "Note " .. idx .. " of " .. #due .. " due")
  else
    table.insert(parts, #due .. " due")
  end
  if #snoozed > 0 then
    table.insert(parts, #snoozed .. " snoozed")
  end
  local meta = quickNotes.frontmatter(page)
  local count = quickNotes.snoozeCount(meta)
  if count > 0 then
    local label = "snoozed " .. count .. "×"
    local at = quickNotes.dueAt(meta)
    if at then
      label = label .. ", back " .. quickNotes.humanDelta(at - os.time())
    end
    table.insert(parts, label)
  end
  table.insert(parts,
    "snooze → " .. quickNotes.humanDelta(quickNotes.nextDelay(count + 1)))
  return table.concat(parts, " · ")
end

local function barButton(label, title, commandName, primary)
  return dom.button {
    class = { "qn-button", primary and "qn-button-primary" },
    title = title,
    onclick = function()
      editor.invokeCommand(commandName)
    end,
    dom.text(label),
  }
end

view.define {
  name = "quickNotes.reviewBar",
  title = "Quick Note Review",
  dock = "page-top",
  supportedDocks = { "page-top", "page-bottom" },
  frame = "minimal",
  defaultOpen = true,
  -- Deliberately not "edit": the status line scans every note, and nothing in
  -- it changes mid-keystroke. A snooze saves the page, which fires "index".
  refreshOn = { "navigate", "index" },
  content = function()
    local page = editor.getCurrentPage()
    if not quickNotes.isNote(page) then
      return nil
    end
    return widget.html(dom.div {
      class = "qn-bar",
      dom.div { class = "qn-status", dom.text(quickNotes.status(page)) },
      dom.div {
        class = "qn-actions",
        barButton("Snooze", "Process later; each snooze brings it back sooner",
          "Quick Notes: Snooze", true),
        barButton("Snooze for…", "Pick when this note comes back",
          "Quick Notes: Snooze For"),
        barButton("Delete", "Delete this quick note",
          "Quick Notes: Delete Current"),
        barButton("‹ Prev", "Previous note due", "Quick Notes: Previous"),
        barButton("Next ›", "Next note due", "Quick Notes: Next"),
      },
    })
  end,
}

-- The queue view ------------------------------------------------------------

view.define {
  name = "quickNotes.queue",
  title = "Quick Notes",
  command = "Quick Notes: Queue",
  key = "Ctrl-q l",
  mac = "Ctrl-q l",
  dock = "modal",
  supportedDocks = { "modal", "lhs", "rhs", "bhs" },
  refreshOn = { "index" },
  placeholder = "Filter quick notes",
  helpText = "Notes waiting for you, then the snoozed ones",
  source = function()
    local due, snoozed = quickNotes.partition()
    local rows = {}
    for _, note in ipairs(due) do
      local count = quickNotes.snoozeCount(note)
      table.insert(rows, {
        name = note.name,
        status = count > 0 and ("due now · snoozed " .. count .. "×")
          or "due now",
        icon = "inbox",
      })
    end
    for _, note in ipairs(snoozed) do
      table.insert(rows, {
        name = note.name,
        status = "back " ..
          quickNotes.humanDelta(quickNotes.dueAt(note) - os.time()) ..
          " · snoozed " .. quickNotes.snoozeCount(note) .. "×",
        icon = "alarm-clock",
      })
    end
    return rows
  end,
  presentation = {
    row = { primary = "name", description = "status", icon = "icon" },
  },
  onSelect = function(row)
    editor.navigate(row.name)
  end,
}
```

```space-style
.qn-bar {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 0.4rem 0.9rem;
  padding: 0.2rem 0;
}

.qn-bar .qn-status {
  color: var(--subtle-color);
  font-size: 0.85em;
}

.qn-bar .qn-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.3rem;
}

.qn-bar .qn-button {
  font: inherit;
  font-size: 0.85em;
  padding: 0.15rem 0.6rem;
  cursor: pointer;
  color: inherit;
  background: transparent;
  border: 1px solid var(--ui-surface-border-color);
  border-radius: 4px;
}

.qn-bar .qn-button:hover {
  border-color: var(--ui-accent-color);
}

.qn-bar .qn-button-primary {
  font-weight: 600;
  border-color: var(--ui-accent-color);
}
```
