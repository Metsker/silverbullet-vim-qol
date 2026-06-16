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

Scrolling is **velocity based**, not step based: holding `j`/`k` ramps to a
fixed cruise speed almost instantly and scrolls smoothly and steadily for as
long as you hold - it never accumulates or accelerates. A quick tap is just a
brief scroll, so it nudges the page a little. Releasing ramps back to a stop.

## Configuration

Set these from your **CONFIG** page (no need to edit this library). Values are
read each time a scroll starts, so changes apply on the next press:

```lua
-- Cruise speed in pixels per second while a key is held (default 1000).
config.set("readOnlyVim.scrollSpeed", 1600)
-- Milliseconds to accelerate to / decelerate from cruise speed (default 60;
-- smaller = snappier start and stop, larger = softer).
config.set("readOnlyVim.scrollRampMs", 60)
```

It also yields to the [Trigger](Trigger.md) overlay, so `j`/`k` still work as
hint letters while hints are showing.

```space-lua
-- priority: 10

local DEFAULT_SPEED = 1000 -- px/s cruise while held; override via config "readOnlyVim.scrollSpeed"
local DEFAULT_RAMP_MS = 60  -- ms to ramp to/from cruise; override via config "readOnlyVim.scrollRampMs"

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

-- One animation frame: ramp the live velocity toward the held direction's cruise
-- speed (or toward 0 when released) and advance the scroller by velocity * dt.
-- dt is real elapsed time, so motion is identical across refresh rates. The
-- next-frame callback indirects through `window` so a reload never stacks loops.
local function animate()
  local scroller = getScroller()
  if not scroller then
    js.window.__sbReadOnlyVimAnimating = false
    return
  end
  local now = js.window.performance.now()
  local dt = (now - js.window.__sbReadOnlyVimLastTime) / 1000
  js.window.__sbReadOnlyVimLastTime = now
  if dt < 0 then
    dt = 0
  end
  if dt > 0.05 then
    dt = 0.05 -- clamp big gaps (background tab) so we never jump
  end

  -- Speed / ramp are captured when the gesture starts (see startLoop).
  local speed = tonumber(js.window.__sbReadOnlyVimSpeed) or DEFAULT_SPEED
  if speed <= 0 then
    speed = DEFAULT_SPEED
  end
  local ramp = tonumber(js.window.__sbReadOnlyVimRamp) or DEFAULT_RAMP_MS
  if ramp < 1 then
    ramp = 1
  end

  local dir = js.window.__sbReadOnlyVimHeldDir or 0
  local targetVel = dir * speed
  local vel = js.window.__sbReadOnlyVimVelocity or 0

  -- Move velocity toward target by at most (speed / ramp) per second.
  local maxDelta = (speed / (ramp / 1000)) * dt
  if vel < targetVel then
    vel = vel + maxDelta
    if vel > targetVel then
      vel = targetVel
    end
  elseif vel > targetVel then
    vel = vel - maxDelta
    if vel < targetVel then
      vel = targetVel
    end
  end

  local pos = scroller.scrollTop + vel * dt
  local max = scroller.scrollHeight - scroller.clientHeight
  if pos <= 0 then
    pos = 0
    vel = 0
  elseif pos >= max then
    pos = max
    vel = 0
  end
  scroller.scrollTop = pos
  js.window.__sbReadOnlyVimVelocity = vel

  -- Keep animating while held, or while coasting to a stop.
  if dir ~= 0 or vel > 0.5 or vel < -0.5 then
    js.window.requestAnimationFrame(function()
      local fn = js.window.__sbReadOnlyVimScrollAnimate
      if fn then
        fn()
      end
    end)
  else
    js.window.__sbReadOnlyVimAnimating = false
  end
end

-- Expose the current frame function so the running loop always reaches the latest
-- definition after a script reload.
js.window.__sbReadOnlyVimScrollAnimate = animate

local function startLoop()
  if not js.window.__sbReadOnlyVimAnimating then
    js.window.__sbReadOnlyVimAnimating = true
    js.window.__sbReadOnlyVimLastTime = js.window.performance.now()
    -- Read config at gesture start (runtime), so CONFIG's config.set has already
    -- run and live edits take effect on the next press. config.get is synchronous.
    js.window.__sbReadOnlyVimSpeed = config.get("readOnlyVim.scrollSpeed", DEFAULT_SPEED)
    js.window.__sbReadOnlyVimRamp = config.get("readOnlyVim.scrollRampMs", DEFAULT_RAMP_MS)
    local fn = js.window.__sbReadOnlyVimScrollAnimate
    if fn then
      fn()
    end
  end
end

local function handleKeyDown(e)
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
  -- Set the held direction; auto-repeat keydowns just re-affirm it (no buildup).
  local dir = 1
  if key == "k" then
    dir = -1
  end
  js.window.__sbReadOnlyVimHeldDir = dir
  js.window.__sbReadOnlyVimHeldKey = key
  startLoop()
end

local function handleKeyUp(e)
  local key = e.key
  if key ~= "j" and key ~= "k" then
    return
  end
  -- Only stop if the released key is the one currently driving the scroll.
  if js.window.__sbReadOnlyVimHeldKey == key then
    js.window.__sbReadOnlyVimHeldDir = 0
    js.window.__sbReadOnlyVimHeldKey = nil
  end
end

-- Expose the current handlers so the permanent bootstrap listeners always reach
-- the latest definitions after a script reload.
js.window.__sbReadOnlyVimScrollHandler = handleKeyDown
js.window.__sbReadOnlyVimScrollKeyupHandler = handleKeyUp

-- Permanent document-level capture listeners for keydown + keyup, plus a blur
-- safety stop. They indirect through the handlers stored on `window`, so reloads
-- don't stack listeners. A fresh guard name ensures the keyup/blur listeners get
-- added even on clients that bootstrapped an earlier (keydown-only) version.
if not js.window.__sbReadOnlyVimBootstrappedV2 then
  js.window.__sbReadOnlyVimBootstrappedV2 = true
  js.window.document.addEventListener("keydown", function(e)
    local h = js.window.__sbReadOnlyVimScrollHandler
    if h then
      h(e)
    end
  end, true)
  js.window.document.addEventListener("keyup", function(e)
    local h = js.window.__sbReadOnlyVimScrollKeyupHandler
    if h then
      h(e)
    end
  end, true)
  -- If the window loses focus mid-hold the keyup may never arrive; stop cleanly.
  js.window.addEventListener("blur", function(e)
    js.window.__sbReadOnlyVimHeldDir = 0
    js.window.__sbReadOnlyVimHeldKey = nil
  end)
end
```
