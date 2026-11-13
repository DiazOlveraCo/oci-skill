# Reliable SSH Every Time — OCI Free Tier Instances

> **Single authoritative workflow** for SSH access to OCI free tier VMs. Follow this exact sequence — it works every time when followed completely.

---

## The Core Problem

OCI free tier instances (including `VM.Standard.E2.1.Micro`, `VM.Standard.A1.Flex`, and other shapes) have **known SSH daemon issues**:
- `sshd.service` fails to start on boot (Oracle documented issue)
- SELinux blocks SSH on Oracle Linux 9
- `oracle-cloud-agent` conflicts with SSH startup
- Insufficient entropy for SSH key generation on micro instances
- Instance may show RUNNING but SSH isn't ready for 30-60s

**Result:** SSH "Connection refused" or "Connection timed out during banner exchange" even when instance shows RUNNING.

---

## OCI CLI Verification (MANDATORY — Run Before Any OCI Command) — PowerShell

**PREREQUISITE: Load PowerShellAI skill first** — `skill("powershellai")` — required for AI-assisted error analysis and robust automation patterns (sync execution, timeouts with kill switch, session persistence, streaming output).

**Before ANY OCI CLI operation (including SSH state checks), verify CLI is installed using the PowerShellAI automation script:**

```powershell
# Use the provided script (recommended) - ABSOLUTE PATH REQUIRED
& "/root/.config/opencode/skills/oci/scripts/Ensure-OCICLI.ps1" -Verbose -PackageManagerTimeout 180 -ScriptTimeout 300

# Quick inline alternative (fallback only)
if (-not (Get-Command oci -ErrorAction SilentlyContinue)) {
    Write-Host "OCI CLI not found. Installing..." -ForegroundColor Yellow
    if (Get-Command dnf) { sudo dnf install -y oracle-cloud-cli }
    elseif (Get-Command apt) { sudo apt update; sudo apt install -y oracle-cloud-cli }
    elseif (Get-Command pacman) { sudo pacman -S --noconfirm oci-cli }
    elseif (Get-Command apk) { sudo apk add --no-cache oci-cli }
    else { bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)" -- --accept-all-defaults }
}
if (Get-Command oci) { Write-Host "OCI CLI ready: $(oci --version)" -ForegroundColor Green }
```

The script (`scripts/Ensure-OCICLI.ps1`) provides **robust automation patterns from powershellai skill**:
- Cross-platform package manager detection (dnf, apt, pacman, apk, yum, zypper)
- Official Oracle installer fallback (auto-cleans existing /root/lib/oracle-cli)
- Post-install verification
- **AI error analysis on EVERY failure** (invoke `ai` to analyze and suggest recovery)
- **Synchronous execution with live output** (no background jobs, users see every progress line)
- **Explicit timeout handling with kill switch** (process-tree kill on timeout)
- **Named PSAI sessions for persistence** (export on failure for debugging)
- **Streaming AI responses** (users see AI analysis in real-time)
- Verbose logging with `-Verbose` flag
- Force reinstall with `-ForceReinstall`

> **⚠️ CRITICAL**: Past sessions proved agents IGNORED this script and ran the Oracle installer directly via `bash -c "$(curl ...)"` and `pwsh -Command "bash ..."` — causing interactive prompt failures and directory conflicts. The `$env:OPENCODE_SKILL_OCI` variable is NOT set. Use the absolute path above. **Do NOT skip loading powershellai skill** — without it, AI error analysis and robust timeout handling are unavailable.

---

## The Reliable Pattern (Proven Working)

### Prerequisites (One-Time Setup)

```bash
# 1. SSH config with correct key (the one that matches instance launch time)
# The Downloads folder had 4 keys; only the 4th (ssh-key-2026-10-31(3).key) was authorized
Host oci oci-micro
    HostName 146.235.239.203
    User opc
    IdentityFile ~/.ssh/oci_vm_key
    IdentitiesOnly yes
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel ERROR
    ConnectTimeout 20
    ServerAliveInterval 30
    ServerAliveCountMax 3

# 2. Verify the key is the correct one
ssh-keygen -lf ~/.ssh/oci_vm_key
# Must match the public key in VM's ~/.ssh/authorized_keys
```

### The Exact Sequence (Every Time)

```bash
# ============================================
# STEP 1: Verify instance is RUNNING (not STOPPING/STARTING)
# ============================================
timeout 10 oci compute instance get --instance-id <INSTANCE_OCID> \
  --query "data.\"lifecycle-state\"" --raw-output --auth security_token
# Must return: RUNNING
# If STOPPING/STARTING/STOPPED → DO NOT PROCEED → Fix state first (see State Management below)

# ============================================
# STEP 2: Wait for SSH readiness (MANDATORY)
# ============================================
# Instance shows RUNNING but sshd isn't ready yet
# Wait 45 seconds minimum after RUNNING state
sleep 45

# ============================================
# STEP 3: Test SSH with short timeout
# ============================================
timeout 15 ssh -o ConnectTimeout=10 -o BatchMode=yes oci "echo SSH_OK"
# If this fails → DO NOT RETRY MORE THAN 2 TIMES → Go to STEP 4

# ============================================
# STEP 4: If SSH fails (max 2 retries), use Serial Console (guaranteed access)
# ============================================
# One-time setup (run once):
timeout 30 oci compute instance-console-connection create \
  --instance-id <INSTANCE_OCID> \
  --ssh-public-key-file ~/.ssh/oci_vm_key.pub \
  --auth security_token \
  --query "data.id" --raw-output

# Connect via serial console:
CONN_ID=$(timeout 15 oci compute instance-console-connection list --instance-id <INSTANCE_OCID> --query "data[0].id" --raw-output --auth security_token)
timeout 30 ssh -i ~/.ssh/oci_vm_key \
  -o ProxyCommand="ssh -W %h:%p -p 443 $CONN_ID@instance-console.us-sanjose-1.oci.oraclecloud.com" \
  -o ConnectTimeout=20 \
  <INSTANCE_OCID>

# ============================================
# STEP 5: In serial console, fix SSH permanently
# ============================================
# Run these commands IN THE SERIAL CONSOLE (as root):
sudo systemctl status sshd
sudo systemctl enable --now sshd
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
sudo dnf install -y haveged && sudo systemctl enable --now haveged
sudo systemctl restart oracle-cloud-agent

# Exit serial console, SSH will now work
exit

# ============================================
# STEP 6: Verify SSH works
# ============================================
timeout 10 ssh oci "echo SSH_OK && hostname && uptime"
# Returns: SSH_OK + instance name + uptime
```

---

## State Management (Fix Before SSH)

### If Instance is STOPPING (stuck >5 min)
```bash
# Force stop
timeout 30 oci compute instance action --instance-id <OCID> --action STOP --auth security_token
# Wait for STOPPED
timeout 15 oci compute instance get --instance-id <OCID> --query "data.\"lifecycle-state\"" --raw-output --auth security_token
```

### If Instance is STOPPED
```bash
# Start
timeout 30 oci compute instance action --instance-id <OCID> --action START --auth security_token
# Wait for RUNNING
timeout 15 oci compute instance get --instance-id <OCID> --query "data.\"lifecycle-state\"" --raw-output --auth security_token
# Then WAIT 45s before SSH (STEP 2 above)
```

### If Instance is RUNNING but SSH fails
```bash
# Soft reset (restarts SSH daemon)
timeout 30 oci compute instance action --instance-id <OCID> --action SOFTRESET --auth security_token
# Wait for RUNNING
timeout 15 oci compute instance get --instance-id <OCID> --query "data.\"lifecycle-state\"" --raw-output --auth security_token
# Then WAIT 45s before SSH (STEP 2 above)
```

---

## Complete One-Command Wrapper

Save as `/usr/local/bin/ssh-oci-reliable` and make executable:

```bash
#!/bin/bash
# ssh-oci-reliable - Guaranteed SSH to OCI free tier instance
# Usage: ssh-oci-reliable [ssh_args...]

set -euo pipefail

INSTANCE_OCID="ocid1.instance.oc1..aaaaaaa..."  # Set your instance OCID
SSH_HOST="oci"  # From ~/.ssh/config
MAX_SSH_RETRIES=2
SSH_TIMEOUT=15
POST_RUNNING_WAIT=45

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() { echo -e "${GREEN}[$(date +'%H:%M:%S')]${NC} $*"; }
warn() { echo -e "${YELLOW}[$(date +'%H:%M:%S')]${NC} $*"; }
err() { echo -e "${RED}[$(date +'%H:%M:%S')]${NC} $*"; }

# Get current state
get_state() {
    timeout 10 oci compute instance get --instance-id "$INSTANCE_OCID" \
        --query "data.\"lifecycle-state\"" --raw-output --auth security_token 2>/dev/null
}

# Wait for specific state
wait_for_state() {
    local target="$1"
    local max_wait=180
    local waited=0
    while [ $waited -lt $max_wait ]; do
        state=$(get_state)
        if [ "$state" = "$target" ]; then
            log "Instance reached $target"
            return 0
        fi
        sleep 5
        waited=$((waited + 5))
    done
    err "Timeout waiting for $target (current: $state)"
    return 1
}

# Test SSH
test_ssh() {
    timeout "$SSH_TIMEOUT" ssh -o ConnectTimeout=10 -o BatchMode=yes "$SSH_HOST" "echo SSH_OK" 2>/dev/null
}

# Serial console fix
fix_via_console() {
    warn "SSH failed, using serial console..."
    
    # Ensure console connection exists
    CONN_ID=$(timeout 15 oci compute instance-console-connection list --instance-id "$INSTANCE_OCID" --query "data[0].id" --raw-output --auth security_token 2>/dev/null)
    if [ -z "$CONN_ID" ] || [ "$CONN_ID" = "null" ]; then
        log "Creating console connection..."
        CONN_ID=$(timeout 30 oci compute instance-console-connection create \
            --instance-id "$INSTANCE_OCID" \
            --ssh-public-key-file ~/.ssh/oci_vm_key.pub \
            --auth security_token \
            --query "data.id" --raw-output)
    fi
    
    log "Connecting to serial console..."
    warn "In console, run the fix commands, then exit"
    
    timeout 60 ssh -i ~/.ssh/oci_vm_key \
        -o ProxyCommand="ssh -W %h:%p -p 443 $CONN_ID@instance-console.us-sanjose-1.oci.oraclecloud.com" \
        -o ConnectTimeout=20 \
        -o StrictHostKeyChecking=no \
        -o UserKnownHostsFile=/dev/null \
        "$INSTANCE_OCID"
}

# Main
log "Checking instance state..."
STATE=$(get_state)
log "Current state: $STATE"

case "$STATE" in
    STOPPING)
        warn "Instance stuck in STOPPING, forcing stop..."
        timeout 30 oci compute instance action --instance-id "$INSTANCE_OCID" --action STOP --auth security_token
        wait_for_state "STOPPED"
        log "Starting instance..."
        timeout 30 oci compute instance action --instance-id "$INSTANCE_OCID" --action START --auth security_token
        wait_for_state "RUNNING"
        ;;
    STOPPED)
        log "Starting instance..."
        timeout 30 oci compute instance action --instance-id "$INSTANCE_OCID" --action START --auth security_token
        wait_for_state "RUNNING"
        ;;
    RUNNING)
        log "Instance is RUNNING"
        ;;
    *)
        err "Unknown state: $STATE"
        exit 1
esac

# Mandatory wait after RUNNING
log "Waiting ${POST_RUNNING_WAIT}s for SSH daemon to be ready..."
sleep "$POST_RUNNING_WAIT"

# Try SSH
for i in $(seq 1 $MAX_SSH_RETRIES); do
    log "SSH attempt $i/$MAX_SSH_RETRIES..."
    if test_ssh; then
        log "SSH connection established!"
        exec ssh "$SSH_HOST" "$@"
    fi
    warn "SSH attempt $i failed"
    sleep 10
done

# Fallback to serial console
fix_via_console

# After console fix, try SSH one more time
log "Retrying SSH after console fix..."
sleep 10
if test_ssh; then
    log "SSH connection established after fix!"
    exec ssh "$SSH_HOST" "$@"
else
    err "SSH still failing after console fix"
    exit 1
fi
```

---

## SSH Config (Must Match)

```ssh
# ~/.ssh/config
Host oci oci-micro
    HostName 146.235.239.203
    User opc
    IdentityFile ~/.ssh/oci_vm_key
    IdentitiesOnly yes
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel ERROR
    ConnectTimeout 20
    ServerAliveInterval 30
    ServerAliveCountMax 3
    TCPKeepAlive yes
```

**Critical:** The `IdentityFile` must be the **exact key authorized on the VM**. In our case, `~/.ssh/oci_vm_key` was copied from `ssh-key-2026-10-31(3).key` (the 4th key in Downloads, matching instance launch time).

---

## Decision Tree (Quick Reference)

```
Instance State?
├── STOPPING (>5 min) → STOP → wait STOPPED → START → wait RUNNING → wait 45s → SSH
├── STOPPED → START → wait RUNNING → wait 45s → SSH
├── RUNNING → wait 45s → SSH
│   ├── SSH works → DONE
│   ├── SSH fails (retry 1) → wait 10s → retry
│   ├── SSH fails (retry 2) → SERIAL CONSOLE → fix sshd + SELinux + haveged → exit → SSH
│   └── SSH fails after console → RECREATE INSTANCE (last resort)
└── TERMINATED → Create new instance with same shape + SSH key
```

---

## Key Timings (Do Not Shorten)

| Step | Duration | Why |
|------|----------|-----|
| Wait after RUNNING | 45s | sshd + oracle-cloud-agent startup |
| SSH ConnectTimeout | 10s | Network latency |
| SSH test timeout | 15s | Banner exchange |
| Console connection | 60s | Serial console handshake |
| State transition wait | 180s max | OCI API eventual consistency |

---

## Verification Checklist (After SSH Works)

```bash
# Run these to verify everything is healthy
ssh oci "systemctl is-active sshd"           # Must return: active
ssh oci "systemctl is-enabled sshd"          # Must return: enabled
ssh oci "getenforce"                         # Must return: Permissive
ssh oci "systemctl is-active haveged"        # Must return: active
ssh oci "systemctl is-active oracle-cloud-agent"  # Must return: active
ssh oci "uptime"                             # Shows uptime since last boot
```

If any check fails → run the serial console fix commands again.

---

## Integration with OCI Skill

**In `stages/02-operations/CONTEXT.md`**, the SSH process MUST follow this exact sequence. No shortcuts, no "try SSH first then figure it out."

**Self-Learning Rule:** If this workflow ever fails, it's a bug in the skill. Create GitHub issue, research root cause, update this file.