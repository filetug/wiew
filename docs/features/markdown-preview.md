# Feature: Markdown Preview

## Summary

Render a local `.md` file in the browser with GitHub-like styling, including live reload when the file changes on disk.

## User Story

> As a developer, I want to run `wiew README.md` and immediately see a rendered, GitHub-styled preview in my browser so I can review documentation without leaving the terminal.

## Acceptance Criteria

- [ ] Running `wiew <file.md>` opens a browser tab with the rendered file
- [ ] Rendering matches GitHub Flavored Markdown (GFM): tables, task lists, fenced code blocks, strikethrough, autolinks
- [ ] Code blocks have syntax highlighting
- [ ] Styling closely matches GitHub's Markdown CSS
- [ ] The browser tab live-reloads within 1 second of the file changing on disk
- [ ] Works on Linux, macOS, and Windows

## Technical Design

### Server

- `net/http` server listens on a random available port (or `--port` override)
- `GET /` — serves the HTML shell
- `GET /content` — returns rendered HTML for the current file
- `GET /ws` — WebSocket for live-reload push notifications
- `GET /assets/*` — serves bundled CSS/JS via `//go:embed`

### Markdown Rendering

Library: `github.com/gomarkdown/markdown`

```
raw bytes → gomarkdown parser (GFM extensions) → HTML string → injected into HTML shell
```

Extensions to enable:
- `Tables`
- `FencedCode`
- `Strikethrough`
- `TaskLists`
- `AutoHeadingIDs`
- `Autolink`

### Syntax Highlighting

Library: `github.com/alecthomas/chroma`

- Applied server-side during HTML generation
- Theme: `github` (light) / `github-dark` (dark, respects `prefers-color-scheme`)

### CSS

- Bundled GitHub Markdown CSS (`github-markdown-light.css`, `github-markdown-dark.css`)
- Embedded via `//go:embed assets/`
- Served from `/assets/`

### Live Reload

- File watcher: `github.com/fsnotify/fsnotify`
- On `Write` or `Rename` event for the watched file:
  1. Re-render Markdown to HTML
  2. Push a `reload` message over the WebSocket
  3. Browser JS receives message and refreshes the content pane (no full page reload)

### Browser Auto-Open

| Platform | Command |
|----------|---------|
| Linux    | `xdg-open` |
| macOS    | `open` |
| Windows  | `start` |

Skipped if `--no-open` flag is set or if a non-local (SSH) session is detected.

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--port` | random | Local server port |
| `--no-open` | false | Skip opening the browser |

## Out of Scope

- Non-Markdown file types (Phase 6)
- Collaborative multi-user preview
- In-browser editing (see [browser-editing.md](browser-editing.md))
