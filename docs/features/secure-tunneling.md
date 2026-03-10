# Feature: Secure Tunneling

## Summary

When running over SSH, expose the local preview server via a temporary, token-gated public URL so the user can open it in their local browser.

## User Story

> As a developer SSHed into a remote server, I want Wiew to print a secure, temporary URL that I can open in my local browser to preview the Markdown file.

## Acceptance Criteria

- [ ] A public HTTPS URL is printed to stdout within 5 seconds of starting Wiew
- [ ] The URL includes a short-lived random token as a URL fragment (`#<token>`)
- [ ] The server validates the token on first browser connection and rejects others
- [ ] The tunnel is torn down when Wiew exits
- [ ] If no supported tunnel backend is found, a clear error message with options is shown
- [ ] The tunnel backend is configurable via `--tunnel` flag
- [ ] Auto-detection falls back gracefully through available backends

## Inline URL Mode (Small Files)

For small files, Wiew can embed the file's metadata and content directly in the URL fragment instead of starting a tunnel. The resulting URL is fully self-contained: `wiew.sh` serves a static page that reads the fragment client-side, decodes the payload, and renders the Markdown — **no local server or tunnel required**.

```
https://wiew.sh/v#<base64url(gzip(JSON payload))>
```

### Payload Schema

```json
{
  "name": "README.md",
  "size": 4821,
  "modified": "2026-03-10T12:00:00Z",
  "content": "# My Project\n..."
}
```

The payload is gzip-compressed then base64url-encoded (RFC 4648 §5, no padding) to keep the fragment as short as possible.

### Size Threshold

| File size (uncompressed) | Behaviour |
|--------------------------|-----------|
| ≤ 50 KB | Inline URL mode (default) |
| > 50 KB | Tunnel mode |

The threshold is configurable via `--inline-limit=<bytes>`. Use `--inline-limit=0` to disable inline mode entirely.

### Why the Fragment?

URL fragments are never transmitted to the server — the payload stays in the browser. `wiew.sh/v` serves a minimal static HTML page; the server never sees the file content. This makes inline URLs safe for content that should not transit a third-party relay.

### Limitations

- URL length: modern browsers support fragments up to ~2 MB; practical Markdown limit is roughly 50 KB uncompressed (~15–20 KB gzipped, ~20–27 KB base64url-encoded)
- Inline URLs cannot support live reload or edit-mode (no persistent connection back to the local machine)
- Not suitable for binary assets embedded in Markdown (images remain hosted elsewhere)

### Inline URL Acceptance Criteria

- [ ] Files ≤ 50 KB produce an inline URL with no tunnel started
- [ ] Payload is gzip-compressed and base64url-encoded
- [ ] `wiew.sh/v` renders the Markdown entirely client-side from the fragment
- [ ] Metadata (filename, size, modified time) is shown in the browser UI
- [ ] Files > 50 KB fall through to tunnel mode automatically
- [ ] `--inline-limit` flag overrides the threshold

---

## Tunnel Backends

For files above the inline size threshold, Wiew supports multiple tunneling backends. See [docs/tunneling-options.md](../tunneling-options.md) for a full comparison with pros and cons.

### Auto-Detection Order

When `--tunnel` is not specified, Wiew probes for available backends in this order:

0. **Inline URL** — if file is within the size threshold (see above)
1. **`cloudflared`** — detected via `exec.LookPath("cloudflared")`
2. **`ngrok`** — detected via `exec.LookPath("ngrok")` + `NGROK_AUTHTOKEN` env var
3. **`localhost.run`** — detected via `exec.LookPath("ssh")`
4. **Wiew relay** (`wiew.sh`) — built-in fallback, always available

### Backend: Cloudflare Tunnel

```
cloudflared tunnel --url http://localhost:<port> --no-autoupdate
```

- Parse the public URL from `cloudflared` stderr (line matching `https://*.trycloudflare.com`)
- No account required

### Backend: ngrok

Uses the official Go SDK (`golang.ngrok.com/ngrok`) — no subprocess needed:

```go
listener, _ := ngrok.Listen(ctx,
    config.HTTPEndpoint(),
    ngrok.WithAuthtoken(os.Getenv("NGROK_AUTHTOKEN")),
)
```

- Requires `NGROK_AUTHTOKEN` environment variable
- URL is available via `listener.URL()`

### Backend: localhost.run (SSH)

```
ssh -o StrictHostKeyChecking=no -R 80:localhost:<port> localhost.run
```

- Parse the assigned URL from SSH stdout (`https://<id>.lhr.rocks`)
- Zero install — uses the system `ssh` binary

### Backend: Wiew Relay

Built-in fallback when no third-party tool is available:

1. Wiew connects to `wss://wiew.sh/relay` over a persistent WebSocket
2. The relay assigns a session ID and public URL: `https://wiew.sh/s/<id>`
3. Browser traffic is proxied through the relay to the local Wiew server

## Token Generation & Validation

```go
token := make([]byte, 16)
rand.Read(token)
sessionToken = hex.EncodeToString(token) // 32 hex chars
```

- Token is embedded in the URL fragment — never transmitted in HTTP requests
- Browser JS reads `location.hash` and sends it as `Authorization: Bearer <token>` on the WebSocket handshake
- Server validates using `subtle.ConstantTimeCompare`; closes the connection on mismatch

## URL Format

```
Inline:         https://wiew.sh/v#<base64url(gzip(JSON))>
Cloudflare:     https://<random>.trycloudflare.com#<token>
ngrok:          https://<random>.ngrok-free.app#<token>
localhost.run:  https://<random>.lhr.rocks#<token>
Wiew relay:     https://wiew.sh/s/<session-id>#<token>
```

## Error Handling

If no backend is available, Wiew exits with a user-friendly error listing options:

```
Error: no tunnel backend found. Options:
  --tunnel=cloudflared   install: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/
  --tunnel=ngrok         install: https://ngrok.com/download  (requires NGROK_AUTHTOKEN)
  --tunnel=localhost.run requires: ssh (usually pre-installed)
  --tunnel=relay         built-in fallback, no install needed
```

## Security Considerations

- The URL fragment (`#<token>`) is never sent to tunnel servers in HTTP requests
- Tokens are cryptographically random (128 bits / 32 hex chars)
- Tunnel sessions are ephemeral — torn down on Wiew exit
- Each session uses a unique token; tokens are not reused

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--tunnel` | auto | Tunnel backend: `cloudflared`, `ngrok`, `localhost.run`, `relay` |
| `--relay-url` | `wss://wiew.sh/relay` | Custom relay WebSocket URL |
| `--inline-limit` | `51200` | Max file size (bytes) for inline URL mode; `0` to disable |

## Out of Scope

- Persistent named tunnels
- Authentication beyond single-use token
- Collaborative multi-user sessions
