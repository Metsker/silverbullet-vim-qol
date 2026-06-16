# SilverBullet Vim QoL

Vim-flavored quality-of-life enhancements for
[SilverBullet](https://silverbullet.md), written in pure
[Space Lua](https://silverbullet.md/Space%20Lua) - no compiled plug, no client
changes. They reach the DOM through the `js` interop bridge and install
document-level `keydown` listeners, so they also work on read-only pages.

Two libraries:

- **Jump** (`Jump.md`) - Vimium-style navigation. Run a command to overlay short
  letter labels on every link, button and task checkbox on screen - plus the
  top-bar actions (page name / rename, home, ...) - then type a label to activate
  it. Great for triggering buttons and links rendered inside query output.
- **Vim Readonly Scroll** (`VimReadonlyScroll.md`) - `j`/`k` scrolling on
  read-only pages while Vim mode is on (where Vim's own motions can't reach the
  unfocusable editor). Inert in edit mode and when Vim is off.

## Install

**Via the library installer (recommended):**

1. In your space, run **`Library: Install`** from the command palette.
2. Paste the URI of the library you want:

   ```
   ghr:Metsker/silverbullet-vim-qol/Jump.md
   ghr:Metsker/silverbullet-vim-qol/VimReadonlyScroll.md
   ```

   (or the GitHub form, e.g. `https://github.com/Metsker/silverbullet-vim-qol/blob/main/Jump.md`)

3. They install as pages under `Library/`. Run **`Library: Update All`** later to
   pull new versions.

**Via a repository (if you want it in the `Libraries: Manager` list):**

1. Run **`Library: Add Repository`** with `ghr:Metsker/silverbullet-vim-qol/Repository.md`.
2. Open **`Libraries: Manager`** and install "Jump".

**Manually:** create a page in your space and paste the contents of
[`Jump.md`](./Jump.md).

## Usage

1. Run **`Navigate: Jump`** (command palette).
2. Yellow letter labels appear on every link, button and task checkbox in view.
3. Type a label to activate it (auto-fires on a unique match).
   - `Backspace` edits your input, `Escape` (or any non-hint key) dismisses.

### Bind it to a key

Add to your `CONFIG` page (use a modifier combo, not a bare letter, so it
doesn't clash with typing):

```lua
command.update {
  name = "Navigate: Jump",
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
- **Text inputs** (the top-bar page-name / rename field) are focused and
  selected, so you can start typing a new name immediately.

## License

MIT
