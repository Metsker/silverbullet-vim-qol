---
name: Library/InputVim
description: Vim insert-mode line editing (Ctrl-W / Ctrl-U / Ctrl-H) in plain text inputs like the command palette
tags: meta/library
---

Vim **insert-mode** line editing in SilverBullet's plain text inputs - the
command palette, the page picker, prompts and the top-bar page-name / rename
field. These inputs aren't CodeMirror, so Vim never sees their keys, and an
unbound `Ctrl-W` falls straight through to the browser (closing the tab). This
library installs a single document-level `keydown` listener that handles the
core Vim insert edits itself and swallows the keys so they never leak.

Provided keys (Ctrl only):

* `Ctrl-W` - delete the word before the cursor
* `Ctrl-U` - delete to the start of the line
* `Ctrl-H` - delete the character before the cursor (Backspace)

The CodeMirror editor and its own panels (search / replace) are deliberately
left alone - the editor already has Vim. Palette navigation (`Ctrl-P` / `Ctrl-N`)
and clipboard keys are untouched too.

Implementation notes: this is pure Space Lua - it reaches the DOM through the
`js` interop bridge (`js.window` is the browser `window`), so it needs no changes
to SilverBullet's client. The command palette is a Preact-controlled `<input>`,
so after rewriting its value the script dispatches an `input` event to make the
palette re-read the text and re-filter, then re-asserts the caret position.

```space-lua
-- priority: 10

local sub = string.sub

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

-- Write a new value + caret into a (framework-controlled) input. The command
-- palette is a Preact-controlled <input>, so we dispatch an "input" event to make
-- it re-read the value and re-filter, then re-assert the caret on the next frame
-- (the controlled re-render writes back the same string and so never moves the
-- cursor - the rAF is just belt-and-suspenders).
local function setValue(input, value, caret)
  input.value = value
  input.dispatchEvent(js.new(js.window.Event, "input", { bubbles = true }))
  input.setSelectionRange(caret, caret)
  js.window.requestAnimationFrame(function()
    input.setSelectionRange(caret, caret)
  end)
end

-- Vim insert-mode line edits, applied only to plain text inputs that are not the
-- CodeMirror editor (which already has Vim) or its panels. Swallows the keys it
-- consumes so they never reach the browser (e.g. Ctrl-W closing the tab).
local function handleKey(e)
  local t = e.target
  if not isTextField(t) then
    return
  end
  if t.closest and t.closest(".cm-editor") then
    return -- leave the editor and its search / replace panels to Vim
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
    local b = before:gsub("%s+$", "")
    b = (b:gsub("%S+$", ""))
    setValue(t, b .. after, #b)
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

-- Expose the current handler so the permanent bootstrap listener can always
-- reach the latest definition, even after scripts are reloaded.
js.window.__sbInputVimHandler = handleKey

-- Install exactly one document-level capture listener for the lifetime of the
-- page. It indirects through the handler stored on `window`, so reloading
-- scripts swaps in fresh logic without stacking listeners.
if not js.window.__sbInputVimBootstrapped then
  js.window.__sbInputVimBootstrapped = true
  js.window.document.addEventListener("keydown", function(e)
    local handler = js.window.__sbInputVimHandler
    if handler then
      handler(e)
    end
  end, true)
end
```
