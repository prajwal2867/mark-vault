# MarkVault

Upload `.md` files and present them as slides in class.

Single-file static prototype. No build step, no backend. Open `index.html` and present.

## Features

- Upload `.md` files via button or drag and drop
- Deck list with search and remove
- Slides split by `---` on its own line
- Slide navigation: Prev/Next buttons, dots, keyboard
- Fullscreen present mode
- Collapsible sidebar (drag edge left to hide, drag right or click Show panel to restore)
- Dark theme, VS Code preview-style typography
- Local persistence via `localStorage` (same browser only)

## Quick Start

1. Open `index.html` in a browser, or serve the folder:
   ```bash
   npx serve .
   ```
2. Click Upload and select one or more `.md` files.
3. Select a deck from the left panel.
4. Navigate slides, then click Present for fullscreen.

No install required. CDN dependencies (`marked`, `highlight.js`) require internet.

## Authoring Slides

Split slides with `---` on its own line:

```markdown
# Title Slide

Intro text here.

---

## Second Slide

- One idea per slide
- Keep bullets short

---

## Code Slide

```python
def hello(name):
    return f"Hello, {name}!"
```
```

If no `---` is present, the whole file renders as one slide.

## Controls

- Next: Right Arrow / Space / Next button
- Previous: Left Arrow / Prev button
- Fullscreen: `F` or Present button, `Esc` to exit
- Hide sidebar: drag divider left past threshold
- Restore sidebar: drag divider right, double-click divider, or click Show panel

## Limits

- `.md` / `.markdown` only, 2 MB per file
- Empty files rejected
- Duplicate names auto-renamed to `name (1).md`
- Storage is per-browser `localStorage`. Clearing site data removes decks. For multi-device sync, see `development plan.md`.

## Project Structure

```
MarkVault/
  index.html           # full app (UI + CSS + JS)
  readme.md            # this file
  development plan.md  # migration path to Vercel multi-client app
```

## Tech

- HTML / CSS / vanilla JS
- `marked` for markdown parsing
- `highlight.js` (vs2015 theme) for code highlighting

## Roadmap

See `development plan.md` for the path to a Vercel-deployable multi-client version (Next.js + Auth + Postgres + Blob storage + shareable present links).
