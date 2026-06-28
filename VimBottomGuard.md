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
* only blocks when the visible cursor is already at the bottom of the rendered
  document content.

```space-lua
-- priority: 20

local CONTENT_BOTTOM_BAND = 40
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

local function getCursor()
  local doc = getDoc()
  return doc.querySelector(".cm-cursor-primary") or doc.querySelector(".cm-cursor") or doc.querySelector(".cm-fat-cursor")
end

local function cursorAtDocumentBottom()
  local cursor = getCursor()
  local content = getContent()
  if not cursor or not content then
    return false
  end

  local cursorRect = cursor.getBoundingClientRect()
  local contentRect = content.getBoundingClientRect()
  return cursorRect.bottom >= contentRect.bottom - CONTENT_BOTTOM_BAND
end

local function noModifiers(e)
  return not (e.ctrlKey or e.metaKey or e.altKey or e.shiftKey)
end

local function rememberGPrefixIfNeeded(e)
  if e.key == "g" and noModifiers(e) and vimActive() and cursorAtDocumentBottom() then
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
  if not cursorAtDocumentBottom() then
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
