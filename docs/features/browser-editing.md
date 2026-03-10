# Feature: Browser Editing

## Summary

Allow users to optionally edit Markdown in the browser, with changes saved back to the local file in real time via WebSocket.

## User Story

> As a developer previewing a Markdown file, I want to optionally switch into edit mode in the browser so I can make quick changes that are immediately saved to my local file.

## Acceptance Criteria

- [ ] Edit mode is opt-in via `--edit` flag
- [ ] The browser renders an editor alongside (or instead of) the preview pane
- [ ] Saving in the browser writes the content to the local file on disk
- [ ] The preview pane updates immediately after a save without a full page reload
- [ ] If the file changes on disk while the browser has unsaved edits, the user is warned
- [ ] Edit mode works over both local and SSH/tunnel connections
- [ ] WebSocket connection is token-gated (same token as the preview URL)

## Technical Design

### Flag

```
wiew --edit README.md
```

When `--edit` is set, the HTML shell switches to a split-pane layout:
- **Left pane**: editor (`<textarea>` or CodeMirror)
- **Right pane**: live rendered preview

### Editor Options

| Option | Notes |
|--------|-------|
| Plain `<textarea>` | Zero JS dependencies; simplest implementation |
| CodeMirror 6 | Syntax highlighting, line numbers; bundled as a single JS file |

Default: plain `<textarea>` for v1; CodeMirror in a future release.

### WebSocket Protocol

Endpoint: `GET /ws`

The browser sends the session token on connection:

```
ws://localhost:<port>/ws?token=<token>
```

#### Message Types (JSON)

**Browser → Server**

```json
{ "type": "save", "content": "# Updated content\n..." }
```

**Server → Browser**

```json
{ "type": "saved", "timestamp": "2026-03-10T12:00:00Z" }
{ "type": "reload", "content": "<rendered HTML>" }
{ "type": "conflict", "diskContent": "# External change\n..." }
```

### Save Flow

1. User clicks **Save** (or `Ctrl+S`) in the browser
2. Browser sends `{ "type": "save", "content": "..." }` over WebSocket
3. Server writes content to the file using `os.WriteFile`
4. Server responds with `{ "type": "saved" }`
5. Server re-renders and pushes `{ "type": "reload", "content": "..." }` to the preview pane

### Conflict Detection

- Server records the file's `ModTime` after each save
- File watcher monitors the file independently
- If an external write changes `ModTime` while the browser has a dirty (unsaved) editor:
  - Server sends `{ "type": "conflict", "diskContent": "..." }`
  - Browser shows a modal: **"File changed on disk — discard your changes or keep them?"**

### Token Gating

- WebSocket upgrade handler reads the `token` query parameter
- Compares using `subtle.ConstantTimeCompare` against the session token
- Returns HTTP 401 on mismatch; closes the connection

## Layout (Edit Mode)

```
+------------------+------------------+
|   Editor (raw)   |  Preview (HTML)  |
|                  |                  |
|  [Save]  [Discard]                  |
+------------------+------------------+
```

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--edit` | false | Enable in-browser editing |

## Out of Scope

- Collaborative multi-user editing
- Full-featured IDE features (autocomplete, linting)
- Version history / undo beyond browser's native undo
