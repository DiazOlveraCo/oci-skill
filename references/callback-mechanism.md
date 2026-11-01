# Browser OAuth Callback Mechanism

## The Flow

```
┌─────────────────┐     1. CLI starts          ┌─────────────────┐
│  OCI CLI        │ ─────────────────────────▶ │  Callback       │
│  (this machine) │   HTTP server on           │  Server         │
│                 │    localhost:8181          │  (this machine) │
└─────────────────┘                            └─────────────────┘
       │                                               │
       │ 2. Opens browser                              │
       ▼                                               │
┌─────────────────┐                                    │
│  Browser        │ 3. User logs in                    │
│  (this machine) │ ──────────────────────────────────▶│
│                 │                                    │
└─────────────────┘                                    │
       │                                               │
       │ 4. OCI redirects to                           │
       │    http://localhost:8181/callback             │
       ▼                                               │
┌─────────────────┐     5. Token exchange            ┌─────────────────┐
│  Callback       │ ◀──────────────────────────────── │  OCI Identity   │
│  Server         │                                  │  Service        │
│  (this machine) │                                  └─────────────────┘
└─────────────────┘
       │
       │ 6. Writes token to ~/.oci/sessions/DEFAULT/
       ▼
┌─────────────────┐
│  Config Updated │
│  ~/.oci/config  │
└─────────────────┘
```

## Critical Requirements

### 1. Browser Must Run on SAME Machine as CLI
- `localhost:8181` callback only works if browser and CLI share network namespace
- **Termux/Android**: Works natively - browser app and Termux share localhost
- **SSH remote**: Requires SSH port forwarding `ssh -L 8181:localhost:8181 remote`
- **Docker**: Requires `-p 8181:8181` port mapping

### 2. xdg-open Must Launch Real Browser

**Termux/Android Default (BROKEN):**
```bash
ls -la $PREFIX/bin/xdg-open
# lrwxrwxrwx ... xdg-open -> termux-open  # Opens file picker, NOT browser
```

**Fix (REQUIRED):**
```bash
ln -sf $PREFIX/bin/termux-open-url $PREFIX/bin/xdg-open
# Now xdg-open → termux-open-url → Android intent → Chrome/Firefox
```

**Verify:**
```bash
xdg-open https://google.com  # Should open browser
```

### 3. CLI Must Stay Running During Auth
- CLI blocks waiting for callback
- Do NOT interrupt with Ctrl+C
- Timeout: ~5-10 minutes default

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Open URL in browser" prints but nothing opens | `xdg-open` broken | `ln -sf termux-open-url xdg-open` |
| Browser opens but callback fails | Browser on different machine | Use SSH tunnel or run CLI locally |
| "Completed browser authentication process!" but no config | Profile name conflict | Use different `--profile-name` |
| Token works once then 401 | Token expired (~60 min) | `oci session authenticate --auth security_token` |

## Manual Callback Test
```bash
# In another terminal, simulate callback
curl "http://localhost:8181/callback?code=TEST&state=TEST"
# Should show CLI receiving callback
```

## Security Note
- Callback server binds to `127.0.0.1:8181` only (not 0.0.0.0)
- Only local processes can reach it
- Token stored in `~/.oci/sessions/DEFAULT/token` (JWT, ~60 min TTL)