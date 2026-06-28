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
* active only for plain `j` or a literal `g` then `j` with no modifiers;
* yields to the Trigger overlay;
* only blocks when the visible cursor or active editor line is already on the
  last rendered line.

```space-lua
-- priority: 20

local LINE_BOTTOM_BAND = 24
local G_PREFIX_MS = 900

local function getDoc()
  return js.window.document
end

local function vimActive()
  return getDoc().querySelector(".cm-vimMode") ~= nil
end

local function getContent()
  return getDoc().querySelector(".cm-content")
end

local function getLastLine()
  local lines = getDoc().querySelectorAll(".cm-line")
  if not lines or lines.length == 0 then
    return nil
  end
  return lines.item(lines.length - 1)
end

local function getActiveLine()
  return getDoc().querySelector(".cm-activeLine")
end

local function getCursor()
  local doc = getDoc()
  return doc.querySelector(".cm-cursor-primary") or doc.querySelector(".cm-cursor") or doc.querySelector(".cm-fat-cursor")
end

local function rectsTouchBottom(a, b)
  if not a or not b then
    return false
  end
  return a.bottom >= b.bottom - LINE_BOTTOM_BAND
end

local function activeLineIsLastLine()
  local activeLine = getActiveLine()
  local lastLine = getLastLine()
  if not activeLine or not lastLine then
    return false
  end
  if activeLine == lastLine then
    return true
  end
  return rectsTouchBottom(activeLine.getBoundingClientRect(), lastLine.getBoundingClientRect())
end

local function cursorAtLastLine()
  local cursor = getCursor()
  local lastLine = getLastLine()
  if not cursor or not lastLine then
    return false
  end
  return rectsTouchBottom(cursor.getBoundingClientRect(), lastLine.getBoundingClientRect())
end

local function atBottomMotionBoundary()
  return activeLineIsLastLine() or cursorAtLastLine()
end

local function noModifiers(e)
  return not (e.ctrlKey or e.metaKey or e.altKey or e.shiftKey)
end

local function rememberGPrefixIfNeeded(e)
  if e.key == "g" and noModifiers(e) and vimActive() and atBottomMotionBoundary() then
    js.window.__sbVimBottomGuardLastG = js.window.performance.now()
  end
end

local function literalGJAtBottom(e)
  if e.key ~= "j" or not noModifiers(e) then
    return false
  end
  local lastG = tonumber(js.window.__sbVimBottomGuardLastG) or 0
  if lastG <= 0 then
    return false
  end
  return js.window.performance.now() - lastG <= G_PREFIX_MS
end

local function mappedJAtBottom(e)
  return e.key == "j" and noModifiers(e)
end

local function shouldBlockDownwardDisplayLine(e)
  if not vimActive() then
    return false
  end
  if getDoc().querySelector(".sb-trigger-hints") then
    return false
  end
  if not atBottomMotionBoundary() then
    return false
  end
  return mappedJAtBottom(e) or literalGJAtBottom(e)
end

local function stopEvent(e)
  e.preventDefault()
  e.stopPropagation()
  if e.stopImmediatePropagation then
    e.stopImmediatePropagation()
  end
end

local function handleKeyDown(e)
  rememberGPrefixIfNeeded(e)
  if shouldBlockDownwardDisplayLine(e) then
    js.window.__sbVimBottomGuardLastG = nil
    stopEvent(e)
  end
end

-- Expose the current handler so reloads update behavior without stacking
-- document listeners.
js.window.__sbVimBottomGuardHandler = handleKeyDown

if not js.window.__sbVimBottomGuardBootstrappedV2 then
  js.window.__sbVimBottomGuardBootstrappedV2 = true
  js.window.addEventListener("keydown", function(e)
    local h = js.window.__sbVimBottomGuardHandler
    if h then
      h(e)
    end
  end, true)
  js.window.document.addEventListener("keydown", function(e)
    local h = js.window.__sbVimBottomGuardHandler
    if h then
      h(e)
    end
  end, true)
end
```
