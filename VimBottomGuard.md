---
name: Library/VimBottomGuard
description: Prevent Vim display-line down movement from wrapping from the bottom of the page back to the top
tags: meta/library
---

Guards a narrow Vim-mode edge case: when `j` is mapped to `gj`, pressing `j`
again at the bottom of a page can make CodeMirror Vim jump back to the top.
This library leaves normal `gj`/`gk` display-line navigation and line wrapping
alone, but suppresses a plain downward `j`/`gj` keypress when the editor is
already at the bottom edge.

It is intentionally conservative:

* active only while Vim mode is enabled;
* active only for plain `j` with no modifiers;
* yields to the Trigger overlay;
* only blocks when the editor scroller is at the bottom and the visible cursor is
  near the bottom of the viewport.

```space-lua
-- priority: 20

local BOTTOM_EPSILON = 3
local CURSOR_BOTTOM_BAND = 80

local function getDoc()
  return js.window.document
end

local function vimActive()
  return getDoc().querySelector(".cm-vimMode") ~= nil
end

local function getScroller()
  return getDoc().querySelector(".cm-scroller")
end

local function scrollerAtBottom(scroller)
  if not scroller then
    return false
  end
  local max = scroller.scrollHeight - scroller.clientHeight
  return scroller.scrollTop >= max - BOTTOM_EPSILON
end

local function cursorNearBottom(scroller)
  local doc = getDoc()
  local cursor = doc.querySelector(".cm-cursor-primary") or doc.querySelector(".cm-cursor") or doc.querySelector(".cm-fat-cursor")
  if not cursor or not scroller then
    return true
  end

  local cursorRect = cursor.getBoundingClientRect()
  local scrollerRect = scroller.getBoundingClientRect()
  return cursorRect.bottom >= scrollerRect.bottom - CURSOR_BOTTOM_BAND
end

local function shouldBlockDownwardDisplayLine(e)
  if e.key ~= "j" then
    return false
  end
  if e.ctrlKey or e.metaKey or e.altKey or e.shiftKey then
    return false
  end
  if getDoc().querySelector(".sb-trigger-hints") then
    return false
  end
  if not vimActive() then
    return false
  end

  local scroller = getScroller()
  return scrollerAtBottom(scroller) and cursorNearBottom(scroller)
end

local function handleKeyDown(e)
  if shouldBlockDownwardDisplayLine(e) then
    e.preventDefault()
    e.stopPropagation()
  end
end

-- Expose the current handler so reloads update behavior without stacking
-- document listeners.
js.window.__sbVimBottomGuardHandler = handleKeyDown

if not js.window.__sbVimBottomGuardBootstrapped then
  js.window.__sbVimBottomGuardBootstrapped = true
  js.window.document.addEventListener("keydown", function(e)
    local h = js.window.__sbVimBottomGuardHandler
    if h then
      h(e)
    end
  end, true)
end
```
