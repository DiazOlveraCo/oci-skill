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
# lrwxrwxrwx ... xdg-open -> termux-open  # Opens file picker, NOT browser (no termux-api)
```

**Fix (REQUIRED):**
```bash
pkg install -y termux-api
ln -sf $PREFIX/bin/termux-open $PREFIX/bin/xdg-open
# Now xdg-open → termux-open → am broadcast → TermuxOpenReceiver → Chrome/Firefox
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
- The OAuth flow works natively when `xdg-open` is linked to `termux-open` (with termux-api)

### ❌ WRONG: "xdg-open is a stub that does nothing"
**This is FALSE.** `xdg-open` works when properly linked to `termux-open` with termux-api.
- Default link: `xdg-open → termux-open` (opens file picker — WRONG without termux-api)
- Fixed link: `xdg-open → termux-open` + termux-api (opens browser — CORRECT)
- Fix: `pkg install -y termux-api && ln -sf $(which termux-open) $(which xdg-open)`

### ❌ WRONG: "No browser exists on this server"
**This is FALSE for Termux/Android.** Termux runs on Android which has Chrome/Firefox/Samsung Internet. The `termux-open` command (with termux-api) launches the Android browser via broadcast.

### ❌ WRONG: "Use termux-open-url for browser OAuth"
**FALSE.** `termux-open-url` uses `am start` which requires `INTERACT_ACROSS_USERS` permission. It FAILS with "Permission Denial: startActivityAsUser...". Use `termux-open` with termux-api instead.

### ✅ CORRECT Understanding
- Termux/Android: Browser OAuth works natively (same network namespace) with termux-api
- SSH remote: Requires SSH port forwarding (`ssh -L 8181:localhost:8181 remote`)
- Docker: Requires port mapping (`-p 8181:8181`)
- API key auth: Alternative for when you want to avoid browser auth entirely

## Security Note
- Callback server binds to `127.0.0.1:8181` only (not 0.0.0.0)
- Only local processes can reach it
- Token stored in `~/.oci/sessions/DEFAULT/token` (JWT, ~60 min TTL)

---

## PROHIBITION: No Shell Scripts for Browser OAuth

**DO NOT create shell scripts, Python scripts, or any executable helpers to automate `oci session authenticate`.**

**Reason:** Shell scripts FAIL to launch the browser link. The OCI CLI prints the auth URL but `xdg-open` inside a **script context** does not reliably open the Android browser. **This is a script-context limitation, not a Termux limitation.**

**VERIFIED WORKING (Interactive Terminal):**
- `xdg-open https://google.com` → opens browser ✅ (exit code 0)
- `oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera` → auto-launches browser ✅
- **Evidence:** Session `ses_e4130dd83ffeZhjOMRKRJ5sP0P` confirmed `xdg-open` works in interactive Termux terminal

**The only reliable way:** Run the command directly in an interactive terminal on the real Termux host (not proot-distro).

---

## ⚠️ CRITICAL: termux-open-url Silent Failure

**`termux-open-url` returns exit code 0 even when it FAILS to open the browser.** This causes OCI CLI to hang forever waiting for a callback that never comes.

**Fix required:** Use the `xdg-open-safe` wrapper (see `references/xdg-open-termux.md`) that forces visible failure by using `-v` flag and checking output for "Activity not started" error.

---

**Required Natural Language Workflow:**

1. **Detect environment** — If in proot-distro/container: browser OAuth will NOT work
2. **Exit to host** — Run `exit` to leave proot-distro, return to real Termux
3. **Fix xdg-open once** — `pkg install -y termux-api && ln -sf $(which termux-open) $(which xdg-open)`
4. **Run auth manually in interactive terminal** — `oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera`
5. **Verify** — `oci os ns get --auth security_token`
6. **Return to proot** — `proot-distro login <distro>` — token is shared via home directory bind mount

This aligns with the Meta skill's "No Helper Scripts — Natural Language Only" principle: skills express logic in plain natural language, not executable code.