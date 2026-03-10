# Wiew — Development Plan

## Goals

Build a Go CLI tool (`wiew`) that lets developers preview and optionally edit Markdown files in a browser, from both local and SSH sessions, using a temporary secure URL.

---

## Phase 1 — Core CLI & Local Server

**Objective:** Get a working local preview for non-SSH use.

### Tasks

- [ ] Initialize Go module (`go mod init github.com/filetug/wiew`)
- [ ] Parse CLI arguments: file path, optional flags (`--no-open`, `--port`, `--edit`)
- [ ] Validate input file exists and is readable
- [ ] Start `net/http` server on a random available port
- [ ] Serve the Markdown file content via `/content` endpoint (raw bytes)
- [ ] Serve a static HTML shell that fetches and renders Markdown
- [ ] Open the browser automatically on local sessions (`xdg-open`, `open`, `start`)
- [ ] Graceful shutdown on `Ctrl+C`

**Deliverable:** `wiew README.md` opens a local browser preview.

---

## Phase 2 — Markdown Rendering

**Objective:** Render Markdown with GitHub-like fidelity.

### Tasks

- [ ] Integrate `github.com/gomarkdown/markdown` for server-side HTML conversion
- [ ] Bundle GitHub Markdown CSS as an embedded asset (`//go:embed`)
- [ ] Support GitHub Flavored Markdown (GFM): tables, task lists, fenced code blocks, strikethrough
- [ ] Add syntax highlighting for code blocks (e.g., `github.com/alecthomas/chroma`)
- [ ] Serve rendered HTML directly; update on file change via polling or `fsnotify`
- [ ] Live-reload the browser tab when the file changes on disk

**Deliverable:** Accurate, styled Markdown rendering with live reload.

---

## Phase 3 — SSH Detection & Secure Tunneling

**Objective:** Work transparently in SSH sessions.

### Tasks

- [ ] Detect SSH session via `$SSH_CLIENT`, `$SSH_TTY`, `$SSH_CONNECTION` env vars
- [ ] If SSH: invoke `cloudflared tunnel --url http://localhost:<port>` as a subprocess
- [ ] Parse the public tunnel URL from `cloudflared` stdout
- [ ] Generate a short-lived random token; append as URL fragment (`#<token>`)
- [ ] Validate the token on first browser connection; reject others
- [ ] Print the full temporary URL to the terminal
- [ ] Fallback: WebSocket relay via `wiew.sh` relay server when `cloudflared` is absent
- [ ] Detect `cloudflared` availability and warn if missing

**Deliverable:** `wiew README.md` over SSH prints a `https://wiew.sh/s/<id>#<token>` URL.

---

## Phase 4 — Browser Editing

**Objective:** Allow optional in-browser Markdown editing that saves back to disk.

### Tasks

- [ ] Add `--edit` flag to enable edit mode
- [ ] Serve a browser-based Markdown editor (e.g., CodeMirror or plain `<textarea>`)
- [ ] Open a WebSocket endpoint (`/ws`) for bidirectional communication
- [ ] On save action in browser: send updated content over WebSocket → write to local file
- [ ] On local file change: push updated content to browser over WebSocket
- [ ] Conflict detection: warn if file changed externally while browser has unsaved edits
- [ ] Token-gate the WebSocket connection (same token as preview URL)

**Deliverable:** `wiew --edit README.md` opens an editable preview; saves go to disk.

---

## Phase 5 — Cross-Platform Packaging & Distribution

**Objective:** Make Wiew easy to install on all platforms.

### Tasks

- [ ] Cross-compile with `GOOS`/`GOARCH` for Linux (amd64, arm64), macOS (amd64, arm64), Windows (amd64)
- [ ] Build `.deb` and `.rpm` packages for Linux
- [ ] Create a Homebrew formula
- [ ] Create a `winget` manifest
- [ ] Set up GitHub Actions CI: build, test, release artifacts on tag push
- [ ] Write install/uninstall scripts
- [ ] Publish release binaries to GitHub Releases

**Deliverable:** One-command install on all major platforms.

---

## Phase 6 — Extensibility & Polish

**Objective:** Prepare for future file-type support and community contributions.

### Tasks

- [ ] Abstract the "renderer" interface so non-Markdown types can be added later
- [ ] Add `--version` flag and build-time version injection
- [ ] Structured logging with log levels (`--verbose`, `--quiet`)
- [ ] Configuration file support (`~/.config/wiew/config.toml`)
- [ ] Man page generation
- [ ] Shell completion (bash, zsh, fish)
- [ ] Contribution guide and issue templates

**Deliverable:** Stable v1.0 ready for open-source release.

---

## Milestone Summary

| Phase | Milestone | Key Output |
|-------|-----------|-----------|
| 1 | Local preview | `wiew file.md` opens browser |
| 2 | Markdown rendering | GitHub-style rendering + live reload |
| 3 | SSH + tunneling | Secure public URL over SSH |
| 4 | Browser editing | In-browser edit → disk save |
| 5 | Packaging | Brew, apt, winget installs |
| 6 | v1.0 | Polished, extensible release |

---

## Feature Specs

Detailed specs for each major feature live in [`docs/features/`](features/).
