---
name: Library/InputVim
description: Vim insert-mode line editing (Ctrl-W / Ctrl-U / Ctrl-H) in text inputs everywhere - command palette, search, Vim command line and panels like the configuration manager
tags: meta/library
---

Vim **insert-mode** line editing in SilverBullet's plain text inputs - everywhere
they appear:

* the command palette and page picker,
* the in-editor **search** box and the Vim **`:` command line**,
* prompts and the top-bar page-name / rename field,
* inputs inside **panels** such as the **configuration manager** (filters,
  search, library forms).

None of these are CodeMirror, so Vim never sees their keys, and an unbound
`Ctrl-W` falls straight through to the browser (closing the tab). This library
handles the core Vim insert edits itself and swallows the keys so they never leak.

Provided keys (Ctrl only):

* `Ctrl-W` - delete the word before the cursor
* `Ctrl-U` - delete to the start of the line
* `Ctrl-H` - delete the character before the cursor (Backspace)

The CodeMirror editor itself is a contenteditable `<div>`, not an `<input>`, so
it's never matched and keeps its own Vim. Palette navigation (`Ctrl-P` /
`Ctrl-N`) and clipboard keys are untouched too.

Implementation notes: this is pure Space Lua, reaching the DOM through the `js`
interop bridge (`js.window` is the browser `window`). A single capture-phase
`keydown` listener on the main document covers the page; panels render in
same-origin `srcdoc` iframes (a separate document), so the script also hooks each
panel iframe's document - discovered with a `MutationObserver` watching only the
direct children of `#sb-root` and `#sb-main`, never the subtree, so CodeMirror's
per-keystroke DOM churn never triggers a rescan. Controlled inputs (the Preact
command palette and configuration manager) are updated by rewriting the value and
dispatching an `input` event, after which the caret is re-asserted.

```space-lua
-- priority: 10

local sub = string.sub

-- True for the whitespace characters that Ctrl-W skips over.
local function isWs(ch)
  return ch == " " or ch == "\t" or ch == "\n" or ch == "\r"
end

-- <input> types we treat as editable text fields.
local textTypes = {
  text = true, search = true, url = true,
  email = true, tel = true, password = true, [""] = true,
}

local function isTextField(t)
  if not t or not t.tagName then
    return false
  end
  if t.tagName == "TEXTAREA" then
    return true
  end
  if t.tagName == "INPUT" then
    return textTypes[t.type] == true
  end
  return false
end

-- Write a new value + caret into a (possibly framework-controlled) input. The
-- input may live in a panel iframe, so use its *own* window for the Event
-- constructor and requestAnimationFrame. Dispatching an "input" event makes a
-- controlled input (the Preact command palette / configuration manager) re-read
-- the value and re-filter; the rAF re-asserts the caret after that re-render
-- (which writes back the same string and so never moves the cursor - belt and
-- suspenders).
local function setValue(input, value, caret)
  local win = input.ownerDocument.defaultView or js.window
  input.value = value
  input.dispatchEvent(js.new(win.Event, "input", { bubbles = true }))
  input.setSelectionRange(caret, caret)
  win.requestAnimationFrame(function()
    input.setSelectionRange(caret, caret)
  end)
end

-- Vim insert-mode line edits for any plain text input. The CodeMirror editor is
-- a contenteditable <div> (not an <input>), so it's never matched and keeps its
-- own Vim; the search box and Vim ":" command line are real <input>s and are
-- handled. Swallows the keys it consumes so they never reach the browser (e.g.
-- Ctrl-W closing the tab).
local function handleKey(e)
  local t = e.target
  if not isTextField(t) then
    return
  end
  -- Ctrl only - no Alt/Meta - so we don't shadow Cmd-W (close window) etc.
  if not (e.ctrlKey and not e.altKey and not e.metaKey) then
    return
  end

  local val = t.value
  local pos = t.selectionStart
  local before = sub(val, 1, pos)
  local after = sub(val, pos + 1)

  if e.code == "KeyW" then
    -- Delete the trailing whitespace + word immediately before the cursor.
    -- Done with sub + length math, not string patterns: Space Lua's pattern
    -- engine errors on end-anchored "%s+$" / "%S+$".
    local n = pos
    while n > 0 and isWs(sub(before, n, n)) do
      n = n - 1
    end
    while n > 0 and not isWs(sub(before, n, n)) do
      n = n - 1
    end
    setValue(t, sub(before, 1, n) .. after, n)
  elseif e.code == "KeyU" then
    -- Delete everything before the cursor.
    setValue(t, after, 0)
  elseif e.code == "KeyH" then
    -- Delete the character before the cursor (Backspace).
    if pos > 0 then
      setValue(t, sub(val, 1, pos - 1) .. after, pos - 1)
    end
  else
    -- Not ours: leave Ctrl-P/N (palette navigation), copy, paste, etc. alone.
    return
  end

  e.preventDefault()
end

-- Expose the current handler so the permanent listeners always reach the latest
-- definition, even after scripts are reloaded.
js.window.__sbInputVimHandler = handleKey

-- Attach one capture-phase keydown listener to a document (the main page or a
-- panel iframe), indirecting through the handler on `window` so a script reload
-- swaps in fresh logic without stacking listeners.
local function attachToDoc(doc)
  if not doc or doc.__sbInputVimAttached then
    return
  end
  doc.__sbInputVimAttached = true
  doc.addEventListener("keydown", function(e)
    local handler = js.window.__sbInputVimHandler
    if handler then
      handler(e)
    end
  end, true)
end

-- Panels (e.g. the configuration manager) render in same-origin srcdoc iframes,
-- so their inputs live in a separate document our main listener can't see. Hook
-- each panel iframe's document - on load, and again immediately in case it
-- already loaded before we noticed it.
local function hookIframe(f)
  if f.__sbInputVimHooked then
    return
  end
  f.__sbInputVimHooked = true
  local function attach()
    local cw = f.contentWindow
    if cw then
      attachToDoc(cw.document)
    end
  end
  f.addEventListener("load", attach)
  attach()
end

local function scanPanels()
  local frames = js.window.document.querySelectorAll(".sb-panel iframe")
  for i = 1, frames.length do
    hookIframe(frames[i])
  end
end

-- One-time bootstrap: listen on the main document, then watch for panels opening
-- so their iframes get hooked. We observe only the *direct children* of #sb-root
-- (modal / bottom panels) and #sb-main (side panels) - childList only, never
-- subtree - so CodeMirror's per-keystroke DOM churn inside #sb-editor never
-- triggers a rescan.
if not js.window.__sbInputVimBootstrapped then
  js.window.__sbInputVimBootstrapped = true
  local doc = js.window.document
  attachToDoc(doc)
  local observer = js.new(js.window.MutationObserver, function()
    scanPanels()
  end)
  local root = doc.getElementById("sb-root")
  local main = doc.getElementById("sb-main")
  if root then
    observer.observe(root, { childList = true })
  end
  if main then
    observer.observe(main, { childList = true })
  end
  if not root and not main then
    observer.observe(doc.body, { childList = true })
  end
  js.window.__sbInputVimObserver = observer
  scanPanels()
end
```
