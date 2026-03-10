# [Wiew.sh](https://wiew.sh)

> Preview and edit Markdown files from the terminal — including over SSH.

**Wiew** is a Go-based CLI tool that starts a local server and exposes a secure temporary URL to a browser, rendering Markdown with GitHub-like styling. It works seamlessly over SSH by detecting remote sessions and creating an ephemeral tunnel automatically.

## Features

- **Markdown Preview** — Render `.md` files in the browser with GitHub-like styling
- **Optional Editing** — Edit Markdown directly in the browser; changes are saved back to your local file
- **CLI-First Workflow** — Simple, one-command usage
- **SSH Friendly** — Detects remote sessions and opens a secure public URL automatically
- **Temporary Secure URLs** — Ephemeral tunnels (Cloudflare Tunnel / ngrok / WebSocket relay) with token-based access
- **Cross-Platform** — Linux, macOS, Windows

## Installation

### macOS
```sh
brew install wiew
```

### Linux
```sh
sudo apt install ./wiew-linux-amd64.deb
```

### Windows
```sh
winget install Wiew
```

### Cloudflared Dependency

Wiew requires `cloudflared` for remote (SSH) previews:

| Platform | Command |
|----------|---------|
| macOS    | `brew install cloudflare/cloudflare/cloudflared` |
| Linux    | Download `.deb` or `.rpm` from [GitHub Releases](https://github.com/cloudflare/cloudflared/releases) |
| Windows  | `winget install Cloudflare.Cloudflared` |

## Usage

```sh
wiew README.md
```

Wiew will:

1. Start a local Go HTTP preview server
2. Detect whether the session is running over SSH
3. Start a secure Cloudflare Tunnel (or WebSocket relay) if needed
4. Print and open a temporary URL in your browser:

```
https://wiew.sh/s/<session>#<token>
```

Optional in-browser editing will save changes back to the local file via WebSocket.

## Technical Overview

| Layer | Technology |
|-------|-----------|
| CLI | Go (`cobra` / `flag`) |
| Local server | Go `net/http` |
| Markdown rendering | `github.com/gomarkdown/markdown` |
| Stylesheet | GitHub Markdown CSS (bundled) |
| Tunneling | Cloudflare Tunnel (`cloudflared`) or WebSocket relay |
| Browser frontend | Plain HTML / CSS / JS |

## Contributing

See [docs/development-plan.md](docs/development-plan.md) for the phased roadmap and [docs/features/](docs/features/) for individual feature specs.

Pull requests are welcome. Please open an issue first to discuss significant changes.

## License

[MIT](LICENSE)
