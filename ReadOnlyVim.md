---
name: Library/ReadOnlyVim
description: Vim quality-of-life helpers for read-only pages (currently j/k scrolling)
tags: meta/library
---

Vim quality-of-life helpers for **read-only** pages. On a read-only page the
editor isn't focusable, so Vim's own motions never receive keys - these install
small document-level listeners that act while the page is read-only *and* Vim is
enabled, and stay completely out of the way in edit mode. More read-only Vim
helpers can be added to this page over time.

Currently provided:

* `j` - scroll down
* `k` - scroll up

Scrolling is **smooth**: each press glides a fixed distance over a fixed time
with an ease-out curve and lands exactly on target (no drift). It's frame-rate
independent - it advances by elapsed time, not per-frame fractions - so it feels
the same on a 60 Hz or 144 Hz display. Holding `j`/`k` tracks the key-repeat for
a continuous cruise; releasing settles within about one step. Two tunables below:

* `SCROLL_STEP` - pixels advanced per key press (bigger = faster cruise).
* `SCROLL_DURATION` - milliseconds to glide one press (smaller = snappier, larger = floatier).

It also yields to the [Trigger](Trigger.md) overlay, so `j`/`k` still work as
hint letters while hints are showing.

```space-lua
-- priority: 10

local SCROLL_STEP = 70      -- pixels advanced per key press (auto-repeats while held)
local SCROLL_DURATION = 140 -- milliseconds to glide one press worth of distance

-- Ease-out cubic: fast start, gentle finish. Frame-rate independent because we
-- feed it elapsed-time fraction t in [0,1], not a per-frame constant.
local function easeOut(t)
  local u = 1 - t
  return 1 - u * u * u
end

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

local function getScroller()
  return js.window.document.querySelector(".cm-scroller")
end

-- One animation frame: interpolate from the press anchor toward the target by
-- the eased time fraction, then either schedule the next frame or land exactly
-- on target. The next-frame callback indirects through `window` so a reload
-- mid-animation never stacks loops.
local function animate()
  local scroller = getScroller()
  local target = js.window.__sbReadOnlyVimScrollTarget
  if not scroller or target == nil then
    js.window.__sbReadOnlyVimScrollAnimating = false
    return
  end
  local start = js.window.__sbReadOnlyVimScrollStart
  local startTime = js.window.__sbReadOnlyVimScrollStartTime
  local duration = js.window.__sbReadOnlyVimScrollDuration
  local elapsed = js.window.performance.now() - startTime
  local t = elapsed / duration
  if t >= 1 then
    -- Land exactly on target and stop - no asymptotic drift.
    scroller.scrollTop = target
    js.window.__sbReadOnlyVimScrollTarget = nil
    js.window.__sbReadOnlyVimScrollAnimating = false
    return
  end
  scroller.scrollTop = start + (target - start) * easeOut(t)
  js.window.requestAnimationFrame(function()
    local fn = js.window.__sbReadOnlyVimScrollAnimate
    if fn then
      fn()
    end
  end)
end

-- Expose the current frame function so the running loop always reaches the latest
-- definition after a script reload.
js.window.__sbReadOnlyVimScrollAnimate = animate

local function scrollBy(delta)
  local scroller = getScroller()
  if not scroller then
    return
  end
  local current = scroller.scrollTop
  -- Extend the in-flight target so repeated presses accumulate (cruise), then
  -- re-anchor the glide to the live position and restart its clock.
  local target = js.window.__sbReadOnlyVimScrollTarget
  if target == nil then
    target = current
  end
  target = target + delta
  -- Clamp to the scrollable range.
  local max = scroller.scrollHeight - scroller.clientHeight
  if target < 0 then
    target = 0
  end
  if target > max then
    target = max
  end
  js.window.__sbReadOnlyVimScrollTarget = target
  js.window.__sbReadOnlyVimScrollStart = current
  js.window.__sbReadOnlyVimScrollStartTime = js.window.performance.now()
  js.window.__sbReadOnlyVimScrollDuration = SCROLL_DURATION
  if not js.window.__sbReadOnlyVimScrollAnimating then
    js.window.__sbReadOnlyVimScrollAnimating = true
    local fn = js.window.__sbReadOnlyVimScrollAnimate
    if fn then
      fn()
    end
  end
end

local function handleScrollKey(e)
  -- Leave keys alone while a Trigger overlay is up, or with modifiers held.
  if js.window.document.querySelector(".sb-trigger-hints") then
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
  e.preventDefault()
  e.stopPropagation()
  if key == "j" then
    scrollBy(SCROLL_STEP)
  else
    scrollBy(-SCROLL_STEP)
  end
end

-- Expose the current handler so the permanent bootstrap listener always reaches
-- the latest definition after a script reload.
js.window.__sbReadOnlyVimScrollHandler = handleScrollKey

-- One document-level capture listener for the lifetime of the page; it indirects
-- through the handler stored on `window`, so reloads don't stack listeners.
if not js.window.__sbReadOnlyVimScrollBootstrapped then
  js.window.__sbReadOnlyVimScrollBootstrapped = true
  js.window.document.addEventListener("keydown", function(e)
    local handler = js.window.__sbReadOnlyVimScrollHandler
    if handler then
      handler(e)
    end
  end, true)
end
```
