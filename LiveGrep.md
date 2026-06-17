---
name: Library/LiveGrep
description: Fuzzy full-text "grep" across every line of every page, jumping straight to the matching line (Telescope live_grep style)
tags: meta/library
---

Fuzzy **full-text search** across your whole space - a Telescope `live_grep` for
SilverBullet. Run **Navigate: Live Grep** (from the Command Palette) to open the
filter box over every line of every page; type to fuzzy-match, then pick a result
to jump straight to that line.

Unlike a tag or object query, it reads page **bodies**, so it also matches text
inside fenced code blocks and frontmatter - not just headers, list items and
paragraphs.

While the filter box is open:

* Type to narrow the list (SilverBullet's built-in fuzzy matching).
* Each result shows the line itself, with `Page:line` underneath.
* `Enter` opens the page at that exact line; `Escape` dismisses.

To bind it to a key, add this to your `CONFIG` page (use a modifier combo so it
doesn't clash with typing):

    command.update {
      name = "Navigate: Live Grep",
      key = "Ctrl-Shift-g",
      mac = "Cmd-Shift-g",
    }

Implementation notes: pure Space Lua, no `js` bridge. It reads each content page
with `space.readPage`, splits it into lines, and feeds the non-blank ones to
`editor.filterBox`; the pick calls `editor.navigate("Page@L<line>C1")`. That ref
is **line based**, so the jump stays correct regardless of multi-byte (e.g.
Cyrillic) characters earlier in the page - a byte offset would drift, a line
number never does.

```space-lua
-- Navigate: Live Grep - fuzzy full-text "grep" across every line of every
-- content page. Reading page bodies (rather than indexed objects) means matches
-- inside code blocks and frontmatter are found too. The filter box does the
-- fuzzy matching; the pick navigates to the exact line via an @L<line> ref.
command.define {
  name = "Navigate: Live Grep",
  run = function()
    local options = {}
    for p in query[[from p = index.contentPages() order by p.name]] do
      local ok, content = pcall(space.readPage, p.name)
      if ok and content then
        local n = 0
        for line in (content .. "\n"):gmatch("([^\n]*)\n") do
          n = n + 1
          local trimmed = string.trim(line)
          if trimmed ~= "" then
            options[#options + 1] = {
              name = trimmed,                   -- shown + fuzzy-matched
              description = p.name .. ":" .. n, -- where the line lives
              page = p.name,
              line = n,
            }
          end
        end
      end
    end
    if #options == 0 then
      editor.flashNotification("No note content to search.", "error")
      return
    end
    local sel = editor.filterBox(
      "Live grep",
      options,
      "Type to fuzzy-search every line in your space"
    )
    if not sel then
      return
    end
    editor.navigate(sel.page .. "@L" .. sel.line .. "C1")
  end
}
```
