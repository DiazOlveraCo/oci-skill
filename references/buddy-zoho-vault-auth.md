# Buddy Automation: Zoho Vault OneAuth (OCI Context)

## Purpose

Automate opening Zoho Vault in Ulaa browser and triggering OneAuth authentication via Buddy MCP. This is a **single-action automation** — open tab + press OneAuth icon + stop. No further interaction.

## Prerequisites

- Buddy MCP server connected (phone at `172.21.154.90:8765`)
- Ulaa browser installed on phone
- Zoho Vault URL accessible
- OneAuth app configured on phone

## Automation Steps (Buddy MCP)

### Step 1: Check for Existing Zoho Vault Tab

Use Buddy screen tools to check if a Zoho Vault tab is already open in Ulaa.

```
buddy.screen_read({})
```

**Look for:** "Zoho Vault" text or Zoho URL in browser tabs.

- **If found:** Skip to Step 3 (OneAuth press).
- **If not found:** Proceed to Step 2 (open tab).

### Step 2: Open Zoho Vault in Ulaa

Use Buddy intent to open Zoho Vault URL in Ulaa browser.

```
buddy.intent({
  action: "android.intent.action.VIEW",
  data: "https://vault.zoho.com",
  flags: ["FLAG_ACTIVITY_NEW_TASK"]
})
```

**Timeout:** 10 seconds. If no response, skip to manual instructions.

**Wait 3 seconds** for page to load.

### Step 3: Press OneAuth Icon

Use Buddy screen tools to find and tap the OneAuth authentication icon.

```
buddy.screen_read({})
```

**Look for:** OneAuth icon/button (typically "OneAuth" or authenticator icon).

If found, tap at its coordinates:

```
buddy.screen_tap({ x: <icon_x>, y: <icon_y> })
```

**Timeout:** 5 seconds. If icon not found or tap fails, skip to manual instructions.

### Step 4: Stop — Do Nothing Else

After pressing OneAuth, **STOP**. Do not:
- Enter any passcodes
- Navigate further
- Interact with any other UI elements
- Read screen again

The OneAuth app will handle the rest (push notification / passcode generation).

## Timeout & Error Handling

| Condition | Action |
|-----------|--------|
| Buddy MCP not connected | Skip automation, instruct user to manually open Zoho Vault + OneAuth |
| Screen read fails (>5s) | Skip automation, instruct user to manually auth |
| OneAuth icon not found | Skip automation, instruct user to manually auth |
| Tap fails | Skip automation, instruct user to manually auth |
| Any buddy error | Skip automation, instruct user to manually auth |

**Total timeout:** 15 seconds for entire automation. If exceeded, abort and instruct user.

## Manual Fallback Instructions

If buddy automation fails or times out:

1. Open **Ulaa browser** on your phone
2. Open **Zoho Vault** (vault.zoho.com)
3. Tap the **OneAuth icon** to authenticate
4. Complete authentication in the OneAuth app
5. Return to Ulaa — Zoho Vault should be authenticated

## Integration with OCI Skill

When the OCI skill is loaded and authentication is needed:

1. **Load buddy skill** in the current OpenCode session
2. **Attempt buddy automation** (Steps 1-4 above) with 15s timeout
3. **If success:** Continue with OCI operations
4. **If failure:** Instruct user to manually auth via Zoho Vault + OneAuth

## Notes

- This automation is **single-purpose**: open tab + press OneAuth + stop
- No passcode entry, no further navigation
- Buddy errors are non-fatal — always fall back to manual instructions
- Timeout prevents hanging on buddy connection issues
- Do NOT retry buddy automation more than once per session