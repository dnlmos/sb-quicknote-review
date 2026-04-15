---
name: "Library/QuickNoteReview"
tags: meta/library
---

# Quick Notes Review

A [SilverBullet](https://silverbullet.md) v2 Space Lua library for reviewing and triaging quick notes stored under `Inbox/`.

When viewing any page under `Inbox/`, three action buttons appear at the top:

**Delete quick note** · **Previous quick note** · **Next quick note**

Run the command **Quick Notes: Review Oldest** from the command palette to jump to the oldest note in `Inbox/` and start walking through them.

Notes are ordered alphabetically by page name (chronological if your filenames are timestamps).

## Implementation

```space-lua
-- priority: 10

quickNotes = quickNotes or {}

-- Return a sorted list of page names under Inbox/ (oldest first, by name).
function quickNotes.list()
  local result = {}
  for _, p in ipairs(space.listPages()) do
    if string.startsWith(p.name, "Inbox/") then
      table.insert(result, p.name)
    end
  end
  table.sort(result)
  return result
end

-- Return 1-based index of `name` in `list`, or nil.
function quickNotes.indexOf(list, name)
  for i, v in ipairs(list) do
    if v == name then return i end
  end
  return nil
end

-- Navigate to the oldest note in Inbox/.
function quickNotes.reviewOldest()
  local notes = quickNotes.list()
  if #notes == 0 then
    editor.flashNotification("No quick notes to review!", "info")
    return
  end
  editor.navigate({ kind = "page", page = notes[1] })
end

-- Navigate to the next note after the current page.
function quickNotes.gotoNext()
  local notes = quickNotes.list()
  local current = editor.getCurrentPage()
  local idx = quickNotes.indexOf(notes, current)
  if idx and idx < #notes then
    editor.navigate({ kind = "page", page = notes[idx + 1] })
  else
    editor.flashNotification("No next quick note.", "info")
  end
end

-- Navigate to the previous note before the current page.
function quickNotes.gotoPrev()
  local notes = quickNotes.list()
  local current = editor.getCurrentPage()
  local idx = quickNotes.indexOf(notes, current)
  if idx and idx > 1 then
    editor.navigate({ kind = "page", page = notes[idx - 1] })
  else
    editor.flashNotification("No previous quick note.", "info")
  end
end

-- Delete the current note and navigate to a sensible next page.
function quickNotes.deleteCurrent()
  local current = editor.getCurrentPage()
  if not string.startsWith(current, "Inbox/") then
    editor.flashNotification("Not a quick note (page isn't under Inbox/).", "error")
    return
  end
  local notes = quickNotes.list()
  local idx = quickNotes.indexOf(notes, current)
  -- Pick where to go after deletion: prefer next, else previous, else index.
  local target = nil
  if idx then
    if idx < #notes then
      target = notes[idx + 1]
    elseif idx > 1 then
      target = notes[idx - 1]
    end
  end
  space.deletePage(current)
  editor.flashNotification("Deleted " .. current, "info")
  if target then
    editor.navigate({ kind = "page", page = target })
  else
    editor.navigate({ kind = "page", page = "index" })
  end
end

-- Commands (callable from the command palette).
command.define {
  name = "Quick Notes: Review Oldest",
  run = quickNotes.reviewOldest
}

command.define {
  name = "Quick Notes: Delete Current",
  run = quickNotes.deleteCurrent
}

command.define {
  name = "Quick Notes: Next",
  run = quickNotes.gotoNext
}

command.define {
  name = "Quick Notes: Previous",
  run = quickNotes.gotoPrev
}

-- Top-of-page action buttons, shown only on pages under Inbox/.
event.listen {
  name = "hooks:renderTopWidgets",
  run = function(e)
    local page = editor.getCurrentPage()
    if not string.startsWith(page, "Inbox/") then return end
    return widget.html(dom.div {
      dom.button {
        onclick = function()
          editor.invokeCommand("Quick Notes: Delete Current")
        end,
        "Delete quick note"
      },
      " | ",
      dom.button {
        onclick = function()
          editor.invokeCommand("Quick Notes: Previous")
        end,
        "Previous quick note"
      },
      " | ",
      dom.button {
        onclick = function()
          editor.invokeCommand("Quick Notes: Next")
        end,
        "Next quick note"
      }
    })
  end
}
```
