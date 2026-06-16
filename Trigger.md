---
name: Library/Trigger
description: Vimium-style keyboard navigation - hint labels to trigger links, buttons, tasks and top-bar actions
tags: meta/library
---

Vimium-style keyboard navigation. Run **Navigate: Trigger** (from the Command
Palette) to overlay short letter labels on every link, button and task checkbox
on screen - plus the **top-bar actions** (page name / rename, home, and the
other action buttons) and the **Table of Contents** widget (its entry links jump
to sections, and its header folds/unfolds it). Type a label to activate it - no
mouse needed. Great for triggering buttons and links rendered inside query output.

While hints are showing:

* Type the letters of a label to activate that element. The hint auto-fires as
  soon as your input uniquely identifies one.
* `Backspace` removes the last typed letter.
* `Escape` (or any non-hint key) dismisses the hints.

Activation dispatches a real click at the element's location (so wiki-link
navigation, external links, widget buttons and task toggles behave exactly as if
clicked), and focuses text inputs like the page-name field so you can rename
straight away.

To bind it to a key, add this to your `CONFIG` page (use a modifier combo, not a
bare letter, so it doesn't clash with typing):

    command.update {
      name = "Navigate: Trigger",
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

-- Everything we consider "clickable", searched within both the top bar and the
-- page content (see collectTargets).
local selector = table.concat({
  "a[href]",
  "a[data-ref]",
  "button",
  "input[type=checkbox]",
  "input.sb-page-name-editor",
  ".sb-task-state[data-task-state]",
  "[role=button]",
  -- Std Table of Contents widget: entry links (<a class=sb-toc-link>, onclick
  -- only, no href) and the foldable header (<summary class=sb-toc-summary>).
  ".sb-toc-link",
  ".sb-toc-summary",
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

-- The opaque chrome that overlaps the scrolling page: the top bar (#sb-top)
-- covers a strip at the top, and sticky bottom panels (the Vim status line,
-- the search box) cover one at the bottom. Returns the top-bar element plus the
-- y-band [topBarBottom, bottomLimit] in which page content is actually visible.
local function chromeLimits()
  local doc = js.window.document
  local topBar = doc.querySelector("#sb-top")
  local topBarBottom = 0
  if topBar then
    local r = topBar.getBoundingClientRect()
    topBarBottom = r.bottom
  end
  local bottomLimit = js.window.innerHeight
  local bottomPanel = doc.querySelector(".cm-panels-bottom")
  if bottomPanel then
    local r = bottomPanel.getBoundingClientRect()
    if r.height > 0 then
      bottomLimit = r.top
    end
  end
  return topBar, topBarBottom, bottomLimit
end

-- Collect visible, clickable elements from the top bar and the page content.
-- The two roots are disjoint, so a hit is in at most one of them.
local function collectTargets()
  local doc = js.window.document
  local viewWidth = js.window.innerWidth
  local viewHeight = js.window.innerHeight
  local topBar, topBarBottom, bottomLimit = chromeLimits()

  -- Each root carries the visible y-band for its targets. The top bar is valid
  -- from the viewport top; page content is only visible between the bar's bottom
  -- and the top of the (sticky) bottom panel - it scrolls behind both.
  local roots = {}
  if topBar then
    roots[#roots + 1] = { el = topBar, top = 0, bottom = viewHeight }
  end
  local content = doc.querySelector(".cm-content")
  if content then
    roots[#roots + 1] = { el = content, top = topBarBottom, bottom = bottomLimit }
  end

  local candidates = {}
  for _, root in ipairs(roots) do
    local nodes = root.el.querySelectorAll(selector)
    for i = 1, nodes.length do
      local el = nodes[i]
      local rect = el.getBoundingClientRect()
      local hasSize = rect.width > 0 or rect.height > 0
      -- Require the element inside the root's visible band, so entries tucked
      -- behind the top bar or the bottom panel are skipped.
      local onScreen = rect.bottom > root.top and rect.right > 0
        and rect.top < root.bottom and rect.left < viewWidth
      -- Skip elements that are laid out but not actually shown - e.g. entries
      -- inside a collapsed <details> (a folded Table of Contents). Those keep a
      -- real bounding rect because the content is hidden via content-visibility
      -- (so it stays text-searchable), not display:none, so the size/on-screen
      -- checks alone don't catch them; checkVisibility() does.
      local visible = true
      if el.checkVisibility then
        visible = el.checkVisibility()
      end
      if hasSize and onScreen and visible then
        candidates[#candidates + 1] = el
      end
    end
  end

  -- Keep the innermost element when candidates nest (e.g. the <button> inside
  -- an <a href> action-button wrapper) so each spot gets a single hint.
  local targets = {}
  for i = 1, #candidates do
    local el = candidates[i]
    local isAncestor = false
    for j = 1, #candidates do
      if i ~= j and el.contains(candidates[j]) then
        isAncestor = true
        break
      end
    end
    if not isAncestor then
      targets[#targets + 1] = el
    end
  end
  return targets
end

local function teardown()
  if session then
    if session.overlay and session.overlay.parentNode then
      session.overlay.parentNode.removeChild(session.overlay)
    end
    session = nil
  end
end

local function fireMouse(el, kind)
  local rect = el.getBoundingClientRect()
  el.dispatchEvent(js.new(js.window.MouseEvent, kind, {
    bubbles = true,
    cancelable = true,
    view = js.window,
    button = 0,
    clientX = rect.left + rect.width / 2,
    clientY = rect.top + rect.height / 2,
  }))
end

-- Activate an element. Links and buttons act on "click" with genuine
-- coordinates (so SilverBullet's own click handling resolves the document
-- position). Task checkboxes toggle on "mouseup" (their "click" handler calls
-- preventDefault); a preceding "mousedown" would make the focused editor
-- re-render and detach the checkbox before the toggle lands, so it is skipped.
-- Text inputs (e.g. the page-name / rename field) are focused for editing.
local function activate(el)
  teardown()
  if el.tagName == "INPUT" then
    if el.type == "checkbox" then
      fireMouse(el, "mouseup")
      fireMouse(el, "click")
    else
      el.focus()
      el.select()
    end
    return
  end
  fireMouse(el, "click")
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
  local targets = collectTargets()
  if #targets == 0 then
    editor.flashNotification("Trigger: nothing clickable in view", "info")
    return
  end

  local doc = js.window.document
  -- Clamp page-content labels into the visible band so a partially-occluded
  -- element's badge stays visible instead of hiding behind the top bar or the
  -- bottom panel.
  local topBar, topBarBottom, bottomLimit = chromeLimits()
  local labels = generateLabels(#targets)
  local overlay = doc.createElement("div")
  overlay.className = "sb-trigger-hints"
  overlay.style.cssText =
    "position:fixed; inset:0; margin:0; padding:0; border:0;" ..
    " z-index:2147483646; pointer-events:none;"

  -- Append the overlay first so each badge's real rendered height can be measured
  -- (rather than hardcoded) when keeping labels clear of the bottom panel.
  doc.body.appendChild(overlay)

  local hints = {}
  local badgeHeight = 0
  for i = 1, #targets do
    local el = targets[i]
    local rect = el.getBoundingClientRect()
    local badge = doc.createElement("div")
    badge.className = "sb-trigger-hint"
    badge.textContent = labels[i]:upper()
    badge.style.cssText =
      "position:fixed; left:" .. px(math.max(0, rect.left)) .. ";" ..
      " background:#ffcf4a; color:#1a1a1a;" ..
      " font:bold 11px/1.1 ui-monospace,SFMono-Regular,Menlo,monospace;" ..
      " padding:1px 4px; border-radius:4px; border:1px solid rgba(0,0,0,0.35);" ..
      " box-shadow:0 1px 3px rgba(0,0,0,0.4); white-space:nowrap;" ..
      " pointer-events:none; z-index:2147483647;"
    overlay.appendChild(badge)
    -- All badges share styling, so measure the height once from the live element.
    if badgeHeight == 0 then
      badgeHeight = badge.offsetHeight
    end
    -- Keep page-content labels within the visible band so they don't hide behind
    -- the top bar or the bottom panel. Top-bar labels keep their position.
    local badgeTop = rect.top
    if topBar and not topBar.contains(el) then
      if badgeTop < topBarBottom then
        badgeTop = topBarBottom
      end
      local maxTop = bottomLimit - badgeHeight
      if badgeTop > maxTop then
        badgeTop = maxTop
      end
    end
    badge.style.top = px(math.max(0, badgeTop))
    hints[#hints + 1] = { el = el, label = labels[i], badge = badge }
  end

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
js.window.__sbTriggerHandler = handleKey

-- Install exactly one document-level capture listener for the lifetime of the
-- page. It indirects through the handler stored on `window`, so reloading
-- scripts swaps in fresh logic without stacking listeners. Capture phase +
-- stopPropagation keeps keys away from CodeMirror while hints are active.
if not js.window.__sbTriggerBootstrapped then
  js.window.__sbTriggerBootstrapped = true
  js.window.document.addEventListener("keydown", function(e)
    local handler = js.window.__sbTriggerHandler
    if handler then
      handler(e)
    end
  end, true)
end

command.define {
  name = "Navigate: Trigger",
  run = showHints,
}
```
