# SilverBullet Link Hints

Vimium-style keyboard navigation for [SilverBullet](https://silverbullet.md).
Run a command to overlay short letter labels on every link, button and task
checkbox on screen, then type a label to activate it - no mouse needed. Great
for triggering buttons and links rendered inside query output.

It's a single [Space Lua](https://silverbullet.md/Space%20Lua) page - no compiled
plug, no client changes. It reaches the DOM through the `js` interop bridge and
installs one document-level `keydown` capture listener, so it also works on
read-only pages.

## Install

**Via the library installer (recommended):**

1. In your space, run **`Library: Install`** from the command palette.
2. Paste the URI:

   ```
   ghr:Metsker/silverbullet-link-hints/LinkHints.md
   ```

   (or the GitHub form `https://github.com/Metsker/silverbullet-link-hints/blob/main/LinkHints.md`)

3. It installs as the page `Library/Link Hints`. Run **`Library: Update All`**
   later to pull new versions.

**Via a repository (if you want it in the `Libraries: Manager` list):**

1. Run **`Library: Add Repository`** with `ghr:Metsker/silverbullet-link-hints/Repository.md`.
2. Open **`Libraries: Manager`** and install "Link Hints".

**Manually:** create a page in your space and paste the contents of
[`LinkHints.md`](./LinkHints.md).

## Usage

1. Run **`Navigate: Link Hints`** (command palette).
2. Yellow letter labels appear on every link, button and task checkbox in view.
3. Type a label to activate it (auto-fires on a unique match).
   - `Backspace` edits your input, `Escape` (or any non-hint key) dismisses.

### Bind it to a key

Add to your `CONFIG` page (use a modifier combo, not a bare letter, so it
doesn't clash with typing):

```lua
command.update {
  name = "Navigate: Link Hints",
  key = "Ctrl-Shift-f",
  mac = "Cmd-Shift-f",
}
```

The combo works on read-only pages too, thanks to SilverBullet's shortcut
forwarding.

## How activation works

Activation dispatches real mouse events at the element's centre:

- **Links / buttons** get a `click` with genuine coordinates, so SilverBullet's
  own click handling (wiki-link navigation, widget buttons) behaves exactly as a
  mouse click would.
- **Task checkboxes** toggle on `mouseup` (their `click` handler calls
  `preventDefault`), so they get `mouseup` + `click` - covering both inline tasks
  and tasks rendered inside query/transclusion output.

## License

MIT
