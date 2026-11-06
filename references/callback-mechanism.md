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

## Common Mistakes (DO NOT REPEAT)

### ❌ WRONG: "Callbacks don't work on Termux/Android"
**This is FALSE.** On Termux/Android:
- Browser app and Termux share the same network namespace
- `localhost:8181` in the browser reaches the CLI's callback server
- The OAuth flow works natively when `xdg-open` is linked to `termux-open-url`

### ❌ WRONG: "xdg-open is a stub that does nothing"
**This is FALSE.** `xdg-open` works when properly linked to `termux-open-url`.
- Default link: `xdg-open → termux-open` (opens file picker — WRONG)
- Fixed link: `xdg-open → termux-open-url` (opens browser — CORRECT)
- Fix: `ln -sf $(which termux-open-url) $(which xdg-open)`

### ❌ WRONG: "No browser exists on this server"
**This is FALSE for Termux/Android.** Termux runs on Android which has Chrome/Firefox/Samsung Internet. The `termux-open-url` command launches the Android browser via intent.

### ✅ CORRECT Understanding
- Termux/Android: Browser OAuth works natively (same network namespace)
- SSH remote: Requires SSH port forwarding (`ssh -L 8181:localhost:8181 remote`)
- Docker: Requires port mapping (`-p 8181:8181`)
- API key auth: Alternative for when you want to avoid browser auth entirely

## Security Note
- Callback server binds to `127.0.0.1:8181` only (not 0.0.0.0)
- Only local processes can reach it
- Token stored in `~/.oci/sessions/DEFAULT/token` (JWT, ~60 min TTL)