# SilverBullet Vim QoL

Vim-flavored quality-of-life enhancements for
[SilverBullet](https://silverbullet.md), written in pure
[Space Lua](https://silverbullet.md/Space%20Lua) - no compiled plug, no client
changes. They reach the DOM through the `js` interop bridge and install
document-level `keydown` listeners, so they also work on read-only pages.

Three libraries:

- **Trigger** (`Trigger.md`) - Vimium-style navigation. Run a command to overlay short
  letter labels on every link, button and task checkbox on screen - plus the
  top-bar actions (page name / rename, home, ...) - then type a label to activate
  it. Great for triggering buttons and links rendered inside query output.
- **ReadOnlyVim** (`ReadOnlyVim.md`) - smooth, velocity-based `j`/`k` scrolling on
  read-only pages while Vim mode is on (where Vim's own motions can't reach the
  unfocusable editor). Holding cruises at a steady speed; a tap nudges a little.
  Inert in edit mode and when Vim is off. Speed is configurable from CONFIG (see below).
- **InputVim** (`InputVim.md`) - Vim insert-mode line editing in plain text inputs:
  the command palette and page picker, the in-editor search box and Vim `:`
  command line, prompts, the rename field and panel search dialogs like
  [SilverSearch](https://github.com/MrMugame/silversearch). `Ctrl-W` (delete word),
  `Ctrl-U` (delete to line start) and `Ctrl-H` (Backspace). These inputs aren't
  CodeMirror, so Vim never sees their keys; this also stops an unbound `Ctrl-W`
  from closing the browser tab. The editor itself keeps its own Vim.

## Install

**Via the library installer (recommended):**

1. In your space, run **`Library: Install`** from the command palette.
2. Paste the URI of the library you want:

   ```
   github:Metsker/silverbullet-vim-qol/Trigger.md
   github:Metsker/silverbullet-vim-qol/ReadOnlyVim.md
   github:Metsker/silverbullet-vim-qol/InputVim.md
   ```

   (or the GitHub form, e.g. `https://github.com/Metsker/silverbullet-vim-qol/blob/main/Trigger.md`)

3. They install as pages under `Library/`. Run **`Library: Update All`** later to
   pull new versions.

**Via a repository (if you want it in the `Libraries: Manager` list):**

1. Run **`Library: Add Repository`** with `github:Metsker/silverbullet-vim-qol/Repository.md`.
2. Open **`Libraries: Manager`** and install the libraries you want.

**Manually:** create a page in your space and paste the contents of
[`Trigger.md`](./Trigger.md).

## Usage

1. Run **`Navigate: Trigger`** (command palette).
2. Yellow letter labels appear on every link, button and task checkbox in view.
3. Type a label to activate it (auto-fires on a unique match).
   - `Backspace` edits your input, `Escape` (or any non-hint key) dismisses.

### Bind it to a key

Add to your `CONFIG` page (use a modifier combo, not a bare letter, so it
doesn't clash with typing):

```lua
command.update {
  name = "Navigate: Trigger",
  key = "Ctrl-Shift-f",
  mac = "Cmd-Shift-f",
}
```

The combo works on read-only pages too, thanks to SilverBullet's shortcut
forwarding.

## ReadOnlyVim scrolling

On a read-only page with Vim mode on, hold `j`/`k` to scroll down/up. It cruises
at a steady speed while held and stops when you release; a quick tap nudges the
page a little. Tune it from your `CONFIG` page (values are read each time a
scroll starts, so edits apply on the next press):

```lua
-- Cruise speed in pixels per second while held (default 1000).
config.set("readOnlyVim.scrollSpeed", 1600)
-- Milliseconds to accelerate to / decelerate from cruise speed (default 60;
-- smaller = snappier start and stop).
config.set("readOnlyVim.scrollRampMs", 60)
```

## InputVim editing keys

Once installed, these Vim insert-mode edits work in SilverBullet's plain text
inputs - the command palette and page picker, the in-editor search box and Vim
`:` command line, prompts, the top-bar page-name / rename field and search boxes
inside panels such as the [SilverSearch](https://github.com/MrMugame/silversearch)
full-text search dialog (the CodeMirror editor itself is a contenteditable, not an
input, so it's left to Vim):

- `Ctrl-W` - delete the word before the cursor
- `Ctrl-U` - delete to the start of the line
- `Ctrl-H` - delete the character before the cursor (Backspace)

There's nothing to configure. As a side effect, `Ctrl-W` in those inputs no
longer falls through to the browser and closes the tab.

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
