# Feature: Secure Tunneling

## Summary

When running over SSH, expose the local preview server via a temporary, token-gated public URL so the user can open it in their local browser.

## User Story

> As a developer SSHed into a remote server, I want Wiew to print a secure, temporary URL that I can open in my local browser to preview the Markdown file.

## Acceptance Criteria

- [ ] A public HTTPS URL is printed to stdout within 5 seconds of starting Wiew
- [ ] The URL includes a short-lived random token as a URL fragment (`#<token>`)
- [ ] The server validates the token on first connection and rejects all others
- [ ] The tunnel is torn down when Wiew exits
- [ ] If `cloudflared` is not installed, a clear error message with install instructions is shown
- [ ] An optional WebSocket relay fallback is available when `cloudflared` is absent

## Technical Design

### Primary Tunnel: Cloudflare Tunnel

Wiew spawns `cloudflared` as a child process:

```
cloudflared tunnel --url http://localhost:<port> --no-autoupdate
```

- Parse the assigned public URL from `cloudflared` stderr (line matching `https://*.trycloudflare.com`)
- Append the session token as a URL fragment: `https://<id>.trycloudflare.com#<token>`
- Print the full URL to stdout
- On Wiew exit: send `SIGTERM` to the `cloudflared` process

### Token Generation & Validation

```go
token := make([]byte, 16)
rand.Read(token)
sessionToken = hex.EncodeToString(token) // 32 hex chars
```

- Token is embedded in the URL fragment (never sent to the tunnel server)
- Browser JS reads `location.hash`, strips `#`, and sends it as `Authorization: Bearer <token>` on the WebSocket handshake or as a query parameter on the first HTTP request
- Server validates on connection; closes the connection on mismatch
- Token is single-use for the WebSocket; for HTTP polling, it is validated on every request

### Fallback: WebSocket Relay

When `cloudflared` is not found in `$PATH`:

1. Wiew connects to the `wiew.sh` relay server over a persistent WebSocket
2. The relay server assigns a session ID and public URL: `https://wiew.sh/s/<id>`
3. Browser traffic is proxied through the relay WebSocket to the local Wiew server
4. The token is appended as a fragment: `https://wiew.sh/s/<id>#<token>`

### Availability Check

```go
_, err := exec.LookPath("cloudflared")
if err != nil {
    // warn and fall back to relay or exit
}
```

If neither `cloudflared` nor relay is available, Wiew exits with a user-friendly error:

```
Error: cloudflared is not installed and no relay is configured.
Install cloudflared: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/
```

## URL Format

```
Primary:  https://<random>.trycloudflare.com#<token>
Relay:    https://wiew.sh/s/<session-id>#<token>
```

## Security Considerations

- The URL fragment (`#<token>`) is never sent to the server in HTTP requests — it stays in the browser
- The browser JS must explicitly read `location.hash` and transmit the token
- Tokens are cryptographically random (128 bits)
- Tunnel URLs are single-use sessions; no persistent routing

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--tunnel` | `cloudflared` | Tunnel backend: `cloudflared` or `relay` |
| `--relay-url` | `wss://wiew.sh/relay` | Custom relay server WebSocket URL |

## Out of Scope

- ngrok support (may be added as a third backend later)
- Persistent named tunnels
- Authentication beyond single-use token
