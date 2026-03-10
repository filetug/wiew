# Feature: Cross-Platform Support

## Summary

Build, package, and distribute Wiew for Linux, macOS, and Windows so users can install it with their native package manager.

## User Story

> As a developer on any platform, I want to install Wiew with a single command using my platform's standard package manager.

## Acceptance Criteria

- [ ] Wiew compiles and runs correctly on Linux (amd64, arm64), macOS (amd64, arm64), and Windows (amd64)
- [ ] Install instructions work without manual PATH configuration
- [ ] A GitHub Actions workflow builds and publishes release artifacts on every version tag
- [ ] `.deb` and `.rpm` packages are provided for Linux
- [ ] A Homebrew formula is available for macOS
- [ ] A `winget` manifest is available for Windows

## Build Matrix

| OS | Arch | Output |
|----|------|--------|
| Linux | amd64 | `wiew-linux-amd64`, `wiew-linux-amd64.deb`, `wiew-linux-amd64.rpm` |
| Linux | arm64 | `wiew-linux-arm64`, `wiew-linux-arm64.deb`, `wiew-linux-arm64.rpm` |
| macOS | amd64 | `wiew-darwin-amd64` |
| macOS | arm64 | `wiew-darwin-arm64` |
| macOS | universal | `wiew-darwin-universal` (lipo of amd64 + arm64) |
| Windows | amd64 | `wiew-windows-amd64.exe` |

## CI/CD: GitHub Actions

### Triggers

- Push to `main` — run tests only
- Push of a version tag (`v*`) — build all targets and publish a GitHub Release

### Workflow Steps

1. `go test ./...`
2. Cross-compile with `GOOS`/`GOARCH` matrix
3. Build `.deb` via `nfpm` or `fpm`
4. Build `.rpm` via `nfpm` or `fpm`
5. Create macOS universal binary with `lipo`
6. Upload all artifacts to GitHub Release
7. Update Homebrew formula in a separate tap repository (via PR)

### Tooling

| Tool | Purpose |
|------|---------|
| `goreleaser` | Cross-compilation orchestration, archive creation, GitHub Release |
| `nfpm` | `.deb` / `.rpm` packaging |
| `lipo` (macOS runner) | Universal binary |

## Packaging Details

### Homebrew (macOS / Linux)

Formula location: `homebrew-tap/Formula/wiew.rb`

```ruby
class Wiew < Formula
  desc "Preview and edit Markdown files from the terminal, including over SSH"
  homepage "https://github.com/filetug/wiew"
  version "1.0.0"
  # sha256 and url populated by goreleaser
end
```

Install:
```sh
brew install filetug/tap/wiew
```

### Debian / Ubuntu (`.deb`)

```sh
sudo apt install ./wiew-linux-amd64.deb
```

Package metadata:
- Binary: `/usr/bin/wiew`
- Man page: `/usr/share/man/man1/wiew.1.gz`

### RPM (Fedora / RHEL)

```sh
sudo rpm -i wiew-linux-amd64.rpm
```

### Windows (`winget`)

Manifest: submitted to `microsoft/winget-pkgs`

```sh
winget install Wiew
```

## Platform-Specific Behaviour

### Browser Open

| Platform | Command |
|----------|---------|
| Linux | `xdg-open <url>` |
| macOS | `open <url>` |
| Windows | `start <url>` (via `cmd /c start`) |

Detection uses `runtime.GOOS`.

### File Paths

- Config: follow `os.UserConfigDir()` — `~/.config/wiew/` on Linux/macOS, `%APPDATA%\wiew\` on Windows
- No hardcoded Unix paths

### Signals

- `SIGTERM` / `SIGINT` handled on Linux and macOS via `os/signal`
- Windows: `os.Interrupt` only (no `SIGTERM`)

## Out of Scope

- 32-bit (x86) targets
- FreeBSD / OpenBSD (may be added later; Go supports them)
- Container image (Docker) — Wiew requires a display; container use is edge-case
