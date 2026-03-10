# Tunneling Options

Wiew needs to expose a local HTTP server as a public HTTPS URL when running over SSH. This document compares available third-party tunneling solutions.

See [features/secure-tunneling.md](features/secure-tunneling.md) for the feature spec and how the chosen backend integrates with Wiew.

---

## Comparison Table

| Solution | Free Tier | Account Required | Install Required | SSH-based | Self-hostable | Custom Domain |
|----------|-----------|-----------------|-----------------|-----------|--------------|---------------|
| **Inline URL** | Yes | No | No | No | Yes | — |
| **Wiew WebSocket Relay** | Yes | No | No | No | Yes | — |
| [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/) | Yes (Quick Tunnels) | No (Quick Tunnels) | Yes (`cloudflared`) | No | Partial | Yes (named tunnels) |
| [ngrok](https://ngrok.com) | Yes (limited) | Yes | Yes | No | No | Paid |
| [localtunnel](https://github.com/localtunnel/localtunnel) | Yes (unlimited) | No | Yes (`npm`) | No | Yes | Subdomain hint |
| [bore](https://github.com/ekzhang/bore) | Yes | No | Yes (binary) | No | Yes | No |
| [serveo](https://serveo.net) | Yes | No | No | Yes | No | Subdomain hint |
| [localhost.run](https://localhost.run) | Yes | No | No | Yes | No | Paid |
| [Pinggy](https://pinggy.io) | Yes (limited) | No (anon) / Yes | No | Yes | No | Paid |
| [Tailscale Funnel](https://tailscale.com/kb/1223/funnel) | Yes | Yes | Yes | No | No | Fixed |
| [zrok](https://zrok.io) | Yes | Yes | Yes (binary) | No | Yes | Yes |
| [PageKite](https://pagekite.net) | Free for OSS | Yes | Yes (Python) | No | Yes | Yes |

---

## Options

### 0. Inline URL (Built-in)

For small files (≤ 50 KB by default), Wiew encodes the file's metadata and content directly into the URL fragment and generates a self-contained `wiew.sh/v#<payload>` link. No tunnel, no local server, no third-party dependency.

```
https://wiew.sh/v#<base64url(gzip({"name":"README.md","content":"...",...}))>
```

The `wiew.sh/v` page is a static HTML file that reads the fragment client-side and renders the Markdown in the browser. The file content is **never transmitted to any server** — it stays entirely in the browser via the URL fragment.

**Pros**
- No install, no account, no running process required
- Instant — URL is generated locally in milliseconds
- Content never leaves the browser (fragment is not sent in HTTP requests)
- Shareable link that works even after Wiew exits
- No dependency on any third-party infrastructure

**Cons**
- Only practical for small files (≤ ~50 KB uncompressed Markdown)
- URL can be long (tens of kilobytes base64url-encoded in the fragment)
- No live reload — link is a snapshot of the file at generation time
- Edit mode not supported (no persistent connection back to local machine)
- Not suitable for files with locally-referenced binary assets (images, etc.)

---

### 1. Wiew WebSocket Relay (Built-in)

A built-in relay server hosted at `wiew.sh` that proxies traffic between the browser and the local Wiew process over a persistent WebSocket connection. Requires no installation and no account — the relay is part of the Wiew infrastructure.

```
wiew (local) ──WebSocket──▶ wiew.sh/relay ◀──HTTPS──▶ browser
```

URL format:
```
https://wiew.sh/s/<session-id>#<token>
```

**Pros**
- No install, no account, no third-party dependency
- Always available as a guaranteed fallback
- Token in URL fragment ensures only the intended recipient can connect
- WebSocket support (required for live reload and edit mode)
- Fully under Wiew's control — no external service dependency
- Self-hostable: operators can point `--relay-url` at their own relay instance

**Cons**
- Relay server must be running and reachable (operational burden on Wiew project)
- All traffic transits the `wiew.sh` relay infrastructure
- Higher latency than direct tunnels (double-hop: local → relay → browser)
- No persistent or named URLs — session ID changes on every run
- Bandwidth costs borne by the Wiew project

---

### 2. [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/) (`cloudflared`)

Quick Tunnels provide an ephemeral public URL with no account or configuration required — just a binary.

**Pros**
- No account needed for ephemeral (Quick) tunnels
- HTTPS by default with Cloudflare's TLS
- High reliability and global CDN
- Free with no rate limits on Quick Tunnels
- WebSocket support
- Active development and wide adoption

**Cons**
- Requires installing the `cloudflared` binary (~30 MB)
- Quick Tunnel URLs are not predictable or persistent
- All traffic passes through Cloudflare's infrastructure (data privacy consideration)
- Subprocess output parsing required to extract the assigned URL (stderr scraping)
- No official Go SDK; must spawn as subprocess

---

### 3. [ngrok](https://ngrok.com)

The most widely known tunneling service, with a polished CLI and official Go SDK.

**Pros**
- Official Go SDK (`golang.ngrok.com/ngrok`) — no subprocess needed; integrates directly in-process
- Stable, predictable API
- Rich feature set: request inspection dashboard, traffic replay, custom headers
- Widely recognized and trusted

**Cons**
- **Free tier requires account** and login (auth token)
- Free tier has connection limits (1 concurrent tunnel, rate limits)
- Paid plan required for custom domains, more connections, and higher bandwidth
- All traffic passes through ngrok's infrastructure
- URLs change on every restart (free tier)

---

### 4. [localtunnel](https://github.com/localtunnel/localtunnel)

Simple open-source tunneling via a Node.js service.

**Pros**
- No account required
- Open-source server — fully self-hostable
- Subdomain hints (`--subdomain`) for friendlier URLs (not guaranteed)
- Free and unlimited on the public instance

**Cons**
- Requires Node.js / npm to install (`npm install -g localtunnel`)
- Public server (`localtunnel.me`) is community-run, less reliable
- No official Go SDK; must spawn as subprocess
- HTTPS certificate management is opaque
- Occasionally throttled or unavailable

---

### 5. [bore](https://github.com/ekzhang/bore)

A minimal, self-hostable reverse proxy tunnel written in Rust.

**Pros**
- Extremely simple and lightweight
- Self-hostable: run your own `bore server` instance
- No account required
- Open-source (MIT)
- Single binary

**Cons**
- No public server with SLA (official instance is best-effort)
- No HTTPS on the tunnel itself — relies on a reverse proxy in front
- Small community; less battle-tested than ngrok or cloudflared
- No Go SDK; subprocess required

---

### 6. [serveo](https://serveo.net)

SSH-based tunneling with no installation required — uses the system's existing SSH client.

**Pros**
- **Zero install** — uses `ssh` which is already present on Linux/macOS
- No account required
- Simple command: `ssh -R 80:localhost:<port> serveo.net`
- Subdomain hints supported

**Cons**
- Depends on an external service with no SLA (intermittently unavailable)
- SSH key required (or password auth, which is less convenient programmatically)
- No WebSocket support guarantee
- Harder to parse assigned URL programmatically from SSH output
- Not self-hostable (closed infrastructure)

---

### 7. [localhost.run](https://localhost.run)

SSH-based tunneling similar to serveo, operated as a commercial service.

**Pros**
- **Zero install** — uses system `ssh`
- No account required for free tier
- More reliable than serveo (commercially operated)
- WebSocket support

**Cons**
- Free tier: random subdomain, 3 simultaneous tunnels, bandwidth limits
- Custom domains require a paid plan
- Traffic passes through their infrastructure
- SSH subprocess management adds complexity

---

### 8. [Pinggy](https://pinggy.io)

SSH-based tunneling with a focus on simplicity and QR code sharing.

**Pros**
- **Zero install** — SSH-based
- No account required for anonymous sessions
- HTTPS by default
- WebSocket and TCP tunnel support

**Cons**
- Anonymous sessions expire after 60 minutes
- Account required for persistent/longer sessions
- Custom domains on paid plans only
- Smaller community and track record than ngrok/cloudflared

---

### 9. [Tailscale Funnel](https://tailscale.com/kb/1223/funnel)

Exposes a local service to the internet via Tailscale's network.

**Pros**
- Deep integration with Tailscale (good for teams already using it)
- HTTPS with managed certificates
- Official Go SDK available

**Cons**
- **Requires a Tailscale account** and the Tailscale daemon to be running
- Not suitable for users without Tailscale
- URL is tied to the machine's Tailscale hostname — not ephemeral
- Overkill for a simple preview use case

---

### 10. [zrok](https://zrok.io)

Open-source self-hostable alternative to ngrok, built on OpenZiti.

**Pros**
- Open-source and self-hostable
- Free public instance available
- Go SDK available
- Persistent share URLs possible
- WebSocket support

**Cons**
- Requires account on public instance
- Smaller community and ecosystem
- More complex setup than simpler options
- Less documentation and fewer examples

---

### 11. [PageKite](https://pagekite.net)

Long-standing tunneling service, popular in the open-source community.

**Pros**
- Self-hostable
- Free for open-source projects
- Stable and proven over many years

**Cons**
- Requires Python and the `pagekite.py` script
- Dated UX and tooling compared to modern alternatives
- Free tier is limited; commercial use requires a paid plan
- No Go SDK

---

## Recommendation for Wiew

For Wiew's use case — ephemeral, no-account-required, zero-friction tunneling — the best options are:

| Priority | Option | Reason |
|----------|--------|--------|
| 0th | **Inline URL** | No install, no account, instant, content stays in browser |
| 1st | **Cloudflare Tunnel** | No account, reliable, HTTPS, WebSocket support |
| 2nd | **ngrok** | Best Go SDK, easiest in-process integration (requires account) |
| 3rd | **serveo / localhost.run** | Zero install fallback using system SSH |
| 4th | **Wiew relay** | Built-in fallback when no third-party tool is available |

The `--tunnel` flag lets users choose their preferred backend. Detection order when `--tunnel` is not set:

0. File ≤ 50 KB → **Inline URL** (no tunnel started)
1. `cloudflared` in `$PATH`
2. `ngrok` in `$PATH` (with `NGROK_AUTHTOKEN` env var)
3. `ssh` available → use `localhost.run`
4. Fall back to built-in `wiew.sh` WebSocket relay
