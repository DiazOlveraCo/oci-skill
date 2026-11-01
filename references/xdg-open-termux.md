# xdg-open Configuration for Termux/Android

## Problem
Default Termux `xdg-open` points to `termux-open` which opens Android's file picker, not a web browser. This breaks the OCI CLI browser OAuth flow.

## Root Cause
```bash
$ ls -la $PREFIX/bin/xdg-open
lrwxrwxrwx ... xdg-open -> termux-open
```

`termux-open` uses `ACTION_VIEW` intent with file MIME type → opens file picker.

## Solution
Link `xdg-open` to `termux-open-url` which uses `ACTION_VIEW` with HTTP URL → opens browser.

```bash
ln -sf $PREFIX/bin/termux-open-url $PREFIX/bin/xdg-open
```

## Verification
```bash
# Should open Chrome/Firefox/Samsung Internet
xdg-open https://login.us-sanjose-1.oraclecloud.com

# Check link
ls -la $PREFIX/bin/xdg-open
# lrwxrwxrwx ... xdg-open -> /data/data/com.termux/files/usr/bin/termux-open-url
```

## Automation (Add to ~/.bashrc)
```bash
# Fix xdg-open for browser OAuth
if [[ -f $PREFIX/bin/termux-open-url ]]; then
    ln -sf $PREFIX/bin/termux-open-url $PREFIX/bin/xdg-open 2>/dev/null
fi
```

## Other Platforms

| Platform | xdg-open Target | Works for OAuth? |
|----------|-----------------|------------------|
| Termux (default) | termux-open | ❌ No |
| Termux (fixed) | termux-open-url | ✅ Yes |
| Linux (GNOME/KDE) | xdg-open (system) | ✅ Yes |
| macOS | open | ✅ Yes |
| WSL | wslview / xdg-open | ✅ Yes |
| SSH remote | N/A | ❌ Needs tunnel |

## Docker Container
```dockerfile
# In Dockerfile
RUN ln -sf /usr/bin/termux-open-url /usr/bin/xdg-open  # If Termux-based
# Or ensure xdg-utils installed
RUN apt-get install -y xdg-utils
```

## Debug Commands
```bash
# Check what xdg-open resolves to
readlink -f $(which xdg-open)

# Test with verbose
xdg-open --help 2>&1 | head -5

# Manual browser launch (fallback)
termux-open-url "https://example.com"
am start -a android.intent.action.VIEW -d "https://example.com"
```