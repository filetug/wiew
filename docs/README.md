# Wiew — Documentation

This directory contains the development plan and feature specifications for Wiew.

## Contents

| File / Directory | Description |
|-----------------|-------------|
| [development-plan.md](development-plan.md) | Phased roadmap with tasks and milestones |
| [tunneling-options.md](tunneling-options.md) | Comparison of tunneling backends with pros and cons |
| [features/](features/) | Individual feature specifications |

## Development Plan

The project is broken into six phases:

1. **Core CLI & Local Server** — `wiew file.md` opens a local browser preview
2. **Markdown Rendering** — GitHub-style rendering with live reload
3. **SSH Detection & Tunneling** — Secure public URL over SSH via Cloudflare Tunnel
4. **Browser Editing** — In-browser editing that saves back to disk
5. **Cross-Platform Packaging** — Homebrew, apt, winget distribution
6. **v1.0 Polish** — Config file, shell completions, man page, extensibility

See [development-plan.md](development-plan.md) for the full breakdown.
