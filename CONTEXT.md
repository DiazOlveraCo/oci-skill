# OCI Workspace Context

## Identity
- **Provider**: Oracle Cloud Infrastructure (OCI)
- **Region**: us-sanjose-1 (primary)
- **Tenancy**: diazolvera
- **Profile**: DEFAULT

## Core Principle: OCI CLI MUST ALWAYS Be Installed

**There is NEVER an acceptable situation where OCI CLI is not installed.** 

Before ANY OCI operation (auth, SSH, instance management, MCP, etc.), the agent MUST:
1. Verify `oci` command is available and working
2. If missing → IMMEDIATELY create a GitHub issue in `DiazOlveraCo/oci-skill` documenting the failure
3. Then IMMEDIATELY install OCI CLI
4. Verify installation succeeds before proceeding

## Authentication — Fresh Auth (One Command)

**CRITICAL: Stop Spamming Auth Links**

```bash
# CHECK FIRST — if token works, DONE
oci os ns get --auth security_token && echo "AUTH OK" && exit 0

# Only if check fails: wipe + fresh auth ONCE
rm -rf /root/.oci/sessions/DEFAULT/token /root/.oci/sessions/DEFAULT/oci_session_token_* 2>/dev/null
oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera
oci os ns get --auth security_token
```

**Once auth is confirmed successful (token exists + verification passes), STOP. Do NOT run `oci session authenticate` again.**

---

## Anti-Patterns (Agent MUST Avoid)

| Anti-Pattern | Correct Behavior |
|--------------|------------------|
| ❌ Agent prints URL and tells user "copy this and open in browser" | ✅ Agent runs command directly; browser auto-launches or URL prints |
| ❌ Agent runs `oci session authenticate` inside proot-distro and waits | ✅ Agent runs command directly; if URL prints, user opens it manually |
| ❌ Agent says "if browser doesn't open, copy the URL manually" | ✅ Agent NEVER suggests manual URL copy; just runs the command |
| ❌ Agent retries same auth command multiple times | ✅ Agent runs fresh auth ONCE |
| ❌ Agent creates shell scripts to automate browser launch | ✅ Agent runs command directly in interactive terminal |
| ❌ **Agent re-runs auth after successful verification** | ✅ **Agent checks first; if `oci os ns get --auth security_token` works, DONE** |

## Current State
- Session token auth at `~/.oci/sessions/DEFAULT/` — **always wipe before use**
- Fresh auth workflow: `rm -rf /root/.oci/sessions/DEFAULT/token /root/.oci/sessions/DEFAULT/oci_session_token_* 2>/dev/null && oci session authenticate ...`
- API keys available for persistent headless auth (`~/.oci/oci_api_key_pkcs1.pem`)
- Buddy MCP integration available for Zoho Vault OneAuth automation

## Callback Works on Termux/Android
- Browser and CLI share same network namespace
- `xdg-open` must be linked to `termux-open-url` (not `termux-open`)
- **This method works natively on Termux/Android**

## API Key Auth (Persistent) — Optional
- Config at `~/.oci/config` with API key
- No expiry (keys persist until revoked)
- Use when you want to avoid browser auth entirely

## Buddy Integration (FALLBACK ONLY)

**Primary: OCI CLI for all operations. Buddy only when CLI fails.**

- Load buddy skill in current OpenCode session for phone automation
- Buddy MCP server: `172.21.154.90:8765` (USB tether) or `127.0.0.1:8765` (local)
- Zoho Vault OneAuth: single action (open tab + press icon + stop), 15s timeout
- On buddy error: fall back to manual instructions (no retries)
- **Use case**: OCI Mobile app instance restart when CLI unavailable

## IMPORTANT: Callback Works on Termux/Android
- Previous incorrect claims that "callbacks don't work" or "localhost:8181 unreachable" were WRONG
- On Termux/Android, browser and CLI share the same network namespace
- `xdg-open` must be linked to `termux-open-url` (not `termux-open`) for browser launch
- The OAuth callback to `localhost:8181` works natively on this platform

## Configuration Files
```
~/.oci/
├── config                          # Main profile (DEFAULT)
└── sessions/
    └── DEFAULT/
        ├── oci_api_key.pem
        ├── oci_api_key_public.pem
        └── token                   # JWT session token
```

## Key Commands Reference

| Operation | Command |
|-----------|---------|
| Verify auth | `oci os ns get --auth security_token` (timeout 10) |
| List compartments | `oci iam compartment list --all --compartment-id-in-subtree true --auth security_token` |
| List instances | `oci compute instance list --compartment-id <tenancy-ocid> --all --auth security_token` |
| List ADs | `oci iam availability-domain list --compartment-id <ocid> --auth security_token` |
| Get images | `oci compute image list --compartment-id <ocid> --shape VM.Standard.A1.Flex --auth security_token` |
| Get subnets | `oci network subnet list --compartment-id <ocid> --auth security_token` |
| Get VNIC details | `oci network vnic get --vnic-id <vnic-ocid> --auth security_token` |

## Current VM
| Property | Value |
|----------|-------|
| Name | instance-20261030-1802 |
| Shape | VM.Standard.E2.1.Micro (AMD) |
| State | RUNNING |
| AD | US-SANJOSE-1-AD-1 |
| Public IP | 146.235.239.203 |
| Private IP | 10.1.21.187 |
| SSH | `ssh oci` or `ssh oci-micro` (alias, Oracle Linux 9, user: opc) |
| Instance OCID | ocid1.instance.oc1..aaaaaaa... (set in reliable-ssh.sh) |
| Authorized Key | `~/.ssh/oci_vm_key` (from ssh-key-2026-10-31(3).key) |

## Reliable SSH Workflow (MANDATORY — See references/reliable-ssh.md)

**Every SSH attempt MUST follow this sequence. No shortcuts.**

```bash
# 1. Verify RUNNING (not STOPPING/STARTING)
timeout 10 oci compute instance get --instance-id $INSTANCE_OCID \
  --query "data.\"lifecycle-state\"" --raw-output --auth security_token
# Must return: RUNNING

# 2. Wait 45s after RUNNING (sshd + oracle-cloud-agent startup)
sleep 45

# 3. Test SSH (max 2 retries)
timeout 15 ssh -o ConnectTimeout=10 -o BatchMode=yes oci "echo SSH_OK"

# 4. If fails (2 retries max) → SERIAL CONSOLE (guaranteed)
# One-time: create console connection
# Then: connect via serial console → fix sshd + SELinux + haveged → exit → SSH works
```

**If instance state != RUNNING:** Fix state first (STOP → START or SOFTRESET) then wait 45s.

See `references/reliable-ssh.md` for complete wrapper script, decision tree, and verification checklist.

See `references/cli-navigation.md` for full CLI navigation guide.

## Environment
```bash
export OCI_CLI_SUPPRESS_FILE_PERMISSIONS_WARNING=True
export OCI_CONFIG_PROFILE=DEFAULT
export OCI_CLI_AUTH=security_token
```