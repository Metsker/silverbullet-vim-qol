---
name: Library/Vim Readonly Scroll
description: j/k scrolling on read-only pages when Vim mode is enabled
tags: meta
---

Adds Vim-style `j` / `k` scrolling on **read-only** pages while **Vim mode** is
enabled. On a read-only page the editor isn't focusable, so Vim's own motions
never receive keys - this installs a small document-level listener that scrolls
the page instead. It only acts when the page is read-only *and* Vim is on, so it
never interferes with typing or with Vim's normal `j`/`k` motion in edit mode.

* `j` - scroll down
* `k` - scroll up

Adjust `SCROLL_STEP` below to taste (pixels per key press; the key auto-repeats
while held). It also yields to the [Link Hints](LinkHints.md) overlay, so `j`/`k`
still work as hint letters while hints are showing.

```space-lua
-- priority: 10

local SCROLL_STEP = 64 -- pixels per key press (auto-repeats while the key is held)

local function vimReadOnlyActive()
  local doc = js.window.document
  local content = doc.querySelector(".cm-content")
  if not content then
    return false
  end
  -- Read-only when the editor content is not editable.
  if content.getAttribute("contenteditable") == "true" then
    return false
  end
  -- Vim mode marks the editor with the cm-vimMode class.
  return doc.querySelector(".cm-vimMode") ~= nil
end

local function handleScrollKey(e)
  -- Leave keys alone while a Link Hints overlay is up, or with modifiers held.
  if js.window.document.querySelector(".sb-link-hints") then
    return
  end
  if e.ctrlKey or e.metaKey or e.altKey then
    return
  end
  local key = e.key
  if key ~= "j" and key ~= "k" then
    return
  end
  if not vimReadOnlyActive() then
    return
  end
  local scroller = js.window.document.querySelector(".cm-scroller")
  if not scroller then
    return
  end
  e.preventDefault()
  e.stopPropagation()
  local delta = SCROLL_STEP
  if key == "k" then
    delta = -SCROLL_STEP
  end
  scroller.scrollBy({ top = delta, left = 0 })
end

-- Expose the current handler so the permanent bootstrap listener always reaches
-- the latest definition after a script reload.
js.window.__sbVimScrollHandler = handleScrollKey

-- One document-level capture listener for the lifetime of the page; it indirects
-- through the handler stored on `window`, so reloads don't stack listeners.
if not js.window.__sbVimScrollBootstrapped then
  js.window.__sbVimScrollBootstrapped = true
  js.window.document.addEventListener("keydown", function(e)
    local handler = js.window.__sbVimScrollHandler
    if handler then
      handler(e)
    end
  end, true)
end
```
