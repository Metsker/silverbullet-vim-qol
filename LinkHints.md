---
name: Library/Link Hints
description: Vimium-style keyboard navigation - hint labels on links, buttons and tasks
tags: meta
---

Vimium-style keyboard navigation. Run **Navigate: Link Hints** (from the
Command Palette) to overlay short letter labels on every link, button and task
checkbox currently on screen. Type a label to activate it - no mouse needed.
This is especially handy for triggering buttons and links rendered inside query
output.

While hints are showing:

* Type the letters of a label to activate that element. The hint auto-fires as
  soon as your input uniquely identifies one.
* `Backspace` removes the last typed letter.
* `Escape` (or any non-hint key) dismisses the hints.

Activation dispatches a real click at the element's location, so wiki-link
navigation, external links, widget buttons and task toggles all behave exactly
as if you had clicked them.

To bind it to a key, add this to your `CONFIG` page (use a modifier combo, not a
bare letter, so it doesn't clash with typing):

    command.update {
      name = "Navigate: Link Hints",
      key = "Ctrl-Shift-f",
      mac = "Cmd-Shift-f",
    }

Implementation notes: this is pure Space Lua - it reaches the DOM through the
`js` interop bridge (`js.window` is the browser `window`), so it needs no changes
to SilverBullet's client. A single document-level `keydown` capture listener
drives selection, which means it also works on read-only pages where the editor
isn't focusable.

```space-lua
-- priority: 10

-- Characters used to build hint labels (home row first for comfort).
local charset = "asdfghjklqwertyuiopzxcvbnm"
local charsetSet = {}
for i = 1, #charset do
  charsetSet[charset:sub(i, i)] = true
end

-- Everything we consider "clickable".
local selector = table.concat({
  "a[href]",
  "a[data-ref]",
  "button",
  "input[type=checkbox]",
  ".sb-task-state[data-task-state]",
  "[role=button]",
}, ", ")

-- The currently active hint session, or nil when no hints are shown.
-- { overlay = <div>, hints = { { el, label, badge } }, typed = "" }
local session = nil

-- Build prefix-free, lowercase hint labels for `n` targets. Short single-letter
-- labels are used while they last, otherwise a breadth-first expansion produces
-- multi-letter labels (e.g. "aa", "ab", ...).
local function generateLabels(n)
  local labels = {}
  if n <= 0 then
    return labels
  end
  local k = #charset
  if n <= k then
    for i = 1, n do
      labels[i] = charset:sub(i, i)
    end
    return labels
  end
  local pool = { "" }
  local offset = 1
  while (#pool - offset + 1) < n do
    local prefix = pool[offset]
    offset = offset + 1
    for i = 1, k do
      pool[#pool + 1] = prefix .. charset:sub(i, i)
    end
  end
  for i = 0, n - 1 do
    labels[i + 1] = pool[offset + i]
  end
  return labels
end

local function px(v)
  return math.floor(v) .. "px"
end

local function teardown()
  if session then
    if session.overlay and session.overlay.parentNode then
      session.overlay.parentNode.removeChild(session.overlay)
    end
    session = nil
  end
end

-- Activate an element by simulating a real mouse interaction at its centre.
-- Genuine coordinates let SilverBullet's own click handler resolve the document
-- position for in-page links. Links and buttons act on "click"; task checkboxes
-- toggle on "mouseup" (their "click" handler calls preventDefault), so
-- checkboxes get the full mousedown/mouseup/click sequence - this covers both
-- inline tasks and tasks rendered inside query output.
local function activate(el)
  teardown()
  local rect = el.getBoundingClientRect()
  local centreX = rect.left + rect.width / 2
  local centreY = rect.top + rect.height / 2
  local function fire(kind)
    el.dispatchEvent(js.new(js.window.MouseEvent, kind, {
      bubbles = true,
      cancelable = true,
      view = js.window,
      button = 0,
      clientX = centreX,
      clientY = centreY,
    }))
  end
  if el.tagName == "INPUT" and el.type == "checkbox" then
    -- Inline tasks toggle on "mouseup". A preceding "mousedown" makes the
    -- focused editor re-render and detach this checkbox before the toggle
    -- lands, so it is deliberately skipped. The trailing "click" covers tasks
    -- rendered inside query/transclusion output, which toggle via the
    -- checkbox's default click instead.
    fire("mouseup")
    fire("click")
  else
    fire("click")
  end
end

-- Hide labels that no longer match the typed prefix.
local function refilter()
  for _, hint in ipairs(session.hints) do
    if hint.label:startsWith(session.typed) then
      hint.badge.style.display = ""
    else
      hint.badge.style.display = "none"
    end
  end
end

local function showHints()
  teardown()
  local doc = js.window.document
  local root = doc.querySelector(".cm-content") or doc.body
  if not root then
    editor.flashNotification("Link Hints: no editor content found", "error")
    return
  end

  local nodes = root.querySelectorAll(selector)
  local viewWidth = js.window.innerWidth
  local viewHeight = js.window.innerHeight
  local targets = {}
  for i = 1, nodes.length do
    local el = nodes[i]
    local rect = el.getBoundingClientRect()
    local hasSize = rect.width > 0 or rect.height > 0
    local onScreen = rect.bottom > 0 and rect.right > 0
      and rect.top < viewHeight and rect.left < viewWidth
    if hasSize and onScreen then
      targets[#targets + 1] = el
    end
  end

  if #targets == 0 then
    editor.flashNotification("Link Hints: nothing clickable in view", "info")
    return
  end

  local labels = generateLabels(#targets)
  local overlay = doc.createElement("div")
  overlay.className = "sb-link-hints"
  overlay.style.cssText =
    "position:fixed; inset:0; margin:0; padding:0; border:0;" ..
    " z-index:2147483646; pointer-events:none;"

  local hints = {}
  for i = 1, #targets do
    local el = targets[i]
    local rect = el.getBoundingClientRect()
    local badge = doc.createElement("div")
    badge.className = "sb-link-hint"
    badge.textContent = labels[i]:upper()
    badge.style.cssText =
      "position:fixed; left:" .. px(math.max(0, rect.left)) ..
      "; top:" .. px(math.max(0, rect.top)) .. ";" ..
      " background:#ffcf4a; color:#1a1a1a;" ..
      " font:bold 11px/1.1 ui-monospace,SFMono-Regular,Menlo,monospace;" ..
      " padding:1px 4px; border-radius:4px; border:1px solid rgba(0,0,0,0.35);" ..
      " box-shadow:0 1px 3px rgba(0,0,0,0.4); white-space:nowrap;" ..
      " pointer-events:none; z-index:2147483647;"
    overlay.appendChild(badge)
    hints[#hints + 1] = { el = el, label = labels[i], badge = badge }
  end

  doc.body.appendChild(overlay)
  session = { overlay = overlay, hints = hints, typed = "" }
end

-- Handle a keystroke while hints are showing. Returns nothing; swallows keys it
-- consumes so they never reach the editor.
local function handleKey(e)
  if not session then
    return
  end
  local key = e.key
  if key == "Shift" or key == "Control" or key == "Alt" or key == "Meta" then
    return
  end
  -- Let genuine modifier shortcuts (Ctrl-..., Cmd-...) through, but exit.
  if e.ctrlKey or e.metaKey or e.altKey then
    teardown()
    return
  end
  if key == "Escape" then
    e.preventDefault()
    e.stopPropagation()
    teardown()
    return
  end
  if key == "Backspace" then
    e.preventDefault()
    e.stopPropagation()
    if #session.typed > 0 then
      session.typed = session.typed:sub(1, -2)
      refilter()
    end
    return
  end
  if #key == 1 then
    local ch = key:lower()
    if charsetSet[ch] then
      local tentative = session.typed .. ch
      local matches = {}
      for _, hint in ipairs(session.hints) do
        if hint.label:startsWith(tentative) then
          matches[#matches + 1] = hint
        end
      end
      -- Always swallow a valid hint character so it can't leak to the editor.
      e.preventDefault()
      e.stopPropagation()
      if #matches == 1 then
        activate(matches[1].el)
      elseif #matches > 1 then
        session.typed = tentative
        local exact = nil
        for _, hint in ipairs(matches) do
          if hint.label == tentative then
            exact = hint
            break
          end
        end
        if exact then
          activate(exact.el)
        else
          refilter()
        end
      end
      -- #matches == 0: ignore the keystroke but stay in hint mode.
      return
    end
  end
  -- Any other key cancels hint mode and is passed through to the editor.
  teardown()
end

-- Expose the current handler so the permanent bootstrap listener (below) can
-- always reach the latest definition, even after scripts are reloaded.
js.window.__sbLinkHintsHandler = handleKey

-- Install exactly one document-level capture listener for the lifetime of the
-- page. It indirects through the handler stored on `window`, so reloading
-- scripts swaps in fresh logic without stacking listeners. Capture phase +
-- stopPropagation keeps keys away from CodeMirror while hints are active.
if not js.window.__sbLinkHintsBootstrapped then
  js.window.__sbLinkHintsBootstrapped = true
  js.window.document.addEventListener("keydown", function(e)
    local handler = js.window.__sbLinkHintsHandler
    if handler then
      handler(e)
    end
  end, true)
end

command.define {
  name = "Navigate: Link Hints",
  run = showHints,
}
```
