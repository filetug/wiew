# Feature: SSH Detection

## Summary

Automatically detect whether Wiew is running inside an SSH session so it can decide whether to open a local browser or generate a secure public URL.

## User Story

> As a developer SSHed into a remote machine, I want Wiew to detect the SSH context automatically and skip trying to open a local browser, without any extra flags.

## Acceptance Criteria

- [ ] SSH session is detected reliably on Linux and macOS
- [ ] Detection does not produce false positives in local terminal emulators
- [ ] When SSH is detected, browser auto-open is suppressed
- [ ] When SSH is detected, the tunneling flow is initiated (see [secure-tunneling.md](secure-tunneling.md))
- [ ] Detection result is logged at `--verbose` level

## Technical Design

### Detection Logic

Check for the presence of any of the following environment variables:

```go
func isSSHSession() bool {
    for _, env := range []string{"SSH_CLIENT", "SSH_TTY", "SSH_CONNECTION"} {
        if os.Getenv(env) != "" {
            return true
        }
    }
    return false
}
```

| Variable | Set by |
|----------|--------|
| `SSH_CLIENT` | OpenSSH client (contains `<remote_ip> <remote_port> <local_port>`) |
| `SSH_TTY` | OpenSSH when a PTY is allocated |
| `SSH_CONNECTION` | OpenSSH (contains full connection info) |

### Edge Cases

| Scenario | Behaviour |
|----------|-----------|
| `tmux` / `screen` inside SSH | Variables inherited — correctly detected as SSH |
| Docker container | No SSH vars set — treated as local |
| VS Code Remote SSH | `SSH_CONNECTION` is set — detected as SSH |
| Manual `export SSH_CLIENT=...` | Treated as SSH (acceptable trade-off) |
| `--force-local` flag | Overrides detection; always treats as local |
| `--force-remote` flag | Overrides detection; always treats as SSH |

## Flags

| Flag | Description |
|------|-------------|
| `--force-local` | Treat as local session regardless of env vars |
| `--force-remote` | Treat as SSH session regardless of env vars |

## Out of Scope

- Detecting Mosh sessions (no reliable env var; treat as local)
- Detecting nested SSH hops
