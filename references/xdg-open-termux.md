# xdg-open Configuration for Termux/Android

## Problem
Default Termux `xdg-open` points to `termux-open` which opens Android's file picker, not a web browser. This breaks the OCI CLI browser OAuth flow.

## Root Cause
```bash
$ ls -la $PREFIX/bin/xdg-open
lrwxrwxrwx ... xdg-open -> termux-open
```

`termux-open` (without termux-api) uses `ACTION_VIEW` intent with file MIME type → opens file picker.

**IMPORTANT: `termux-open-url` is BROKEN.** It uses `am start` which requires `INTERACT_ACROSS_USERS` permission. It FAILS with "Permission Denial: startActivityAsUser...". DO NOT USE IT.

---

## Solution (TWO STEPS REQUIRED)

**Step 1:** Install termux-api package
```bash
pkg install -y termux-api
```

**Step 2:** Link `xdg-open` to `termux-open` (NOT termux-open-url!)
```bash
ln -sf $PREFIX/bin/termux-open $PREFIX/bin/xdg-open
```

`termux-open` (with termux-api) uses `am broadcast` to `com.termux/com.termux.app.TermuxOpenReceiver` → opens browser (Chrome, Firefox, Samsung Internet, etc.).

---

## Verification (Mandatory After Fix)

```bash
# 1. Check the link
ls -la $(which xdg-open)
# lrwxrwxrwx ... xdg-open -> /data/data/com.termux/files/usr/bin/termux-open

# 2. Check termux-api is installed
pkg list-installed termux-api

# 3. Test browser launch
xdg-open https://google.com
# Should open Chrome/Firefox/Samsung Internet on the Android device

# 4. Test with OAuth-style URL
xdg-open "https://login.us-sanjose-1.oraclecloud.com/v1/oauth2/authorize?client_id=..."
# Should open browser to OCI login page
```

**If the browser opens → the fix works.** If file picker opens → termux-api not installed or link is wrong.

---

## Automation (Add to ~/.bashrc)

```bash
# Fix xdg-open for browser OAuth on Termux startup
if [[ -f $PREFIX/bin/termux-open ]]; then
    ln -sf $PREFIX/bin/termux-open $PREFIX/bin/xdg-open 2>/dev/null
fi
```

**Requires:** `termux-api` package (`pkg install termux-api`)

---

## Other Platforms

| Platform | xdg-open Target | Works for OAuth? |
|----------|-----------------|------------------|
| Termux (default) | termux-open | ❌ No — opens file picker |
| Termux (fixed) | termux-open (with termux-api) | ✅ Yes — opens browser via broadcast |
| Linux (GNOME/KDE) | system xdg-open | ✅ Yes |
| macOS | open | ✅ Yes |
| WSL | wslview / xdg-open | ✅ Yes |
| SSH remote | N/A | ❌ Needs SSH tunnel |
| Docker container | varies | ⚠️ Depends on setup |

---

## Docker / proot Container Note

If running Termux inside a proot container (proot-distro) or Docker:

```dockerfile
# In Dockerfile for Termux-based image
RUN ln -sf /usr/bin/termux-open /usr/bin/xdg-open  # If Termux-based with termux-api
# Or ensure xdg-utils installed
RUN apt-get install -y xdg-utils
```

**But**: In a container without Android display, `termux-open` will not work (no Android broadcast receiver). See `references/proot-containers.md` and `references/browser-oauth-termux.md` for headless handling.

---

## Debug Commands

```bash
# Check what xdg-open resolves to
readlink -f $(which xdg-open)

# Test browser launch
xdg-open https://google.com

# Manual browser launch (WORKING method)
termux-open "https://example.com"

# Check if termux-open exists and termux-api is installed
which termux-open
ls -la $PREFIX/bin/termux-open
pkg list-installed termux-api
```

---

## ⚠️ CRITICAL: termux-open-url is BROKEN

**`termux-open-url` uses `am start` which requires `android.permission.INTERACT_ACROSS_USERS_FULL`.** This permission is NOT granted to the Termux app. It FAILS with:

```
Permission Denial: startActivityAsUser asks to run as user -2 but is calling from uid u0a326; this requires android.permission.INTERACT_ACROSS_USERS_FULL
```

**DO NOT USE termux-open-url.** It will never work on standard Android.

---

## ⚠️ THE CLAIM "LINKS CAN'T BE OPENED IN TERMUX" IS FALSE

**See dedicated page: `xdg-open skill → references/LINKS_CAN_BE_OPENED.md`**

| Environment | `xdg-open` Works? | Evidence |
|-------------|-------------------|----------|
| **Real Termux host** (with termux-api) | ✅ YES | Verified working |
| **proot-distro** | ❌ No display | Container limitation, not Termux |

**If you're in proot-distro, you are NOT in Termux for browser purposes.** The limitation is the container, not Termux.