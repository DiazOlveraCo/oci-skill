---
name: oci
description: LOAD THIS SKILL IMMEDIATELY when the user says "use oci skill", "run oci skill", "load oci skill", or requests Oracle Cloud Infrastructure operations, CLI commands, authentication, instance management, VCN/networking, SSH access, MCP integration, or capacity automation. FRESH LOAD every explicit request, even if already loaded. For implicit triggers (OCI-related tasks, VM access, security lists): load when needed, reuse if available.
---

# OCI Skill for OpenCode

**Load Behavior**: Agent goes idle on load — no summary, no overview. (Per Meta skill: Skill Load Behavior — Idle on Load)

**Purpose**: Work with Oracle Cloud Infrastructure using CLI, MCP, and API operations following ICM (Interpretable Context Methodology).

---

## Five-Layer Context Hierarchy

| Layer | File | Purpose |
|-------|------|---------|
| 0 | `SKILL.md` | Entry point - routes to stages |
| 1 | `CONTEXT.md` | Workspace-level context (this file) |
| 2 | `stages/01-auth/CONTEXT.md` | Auth stage contract |
| 3 | `stages/02-operations/CONTEXT.md` | Operations stage contract |
| 4 | `stages/03-mcp/CONTEXT.md` | MCP integration stage contract |
| 3 (factory) | `_config/` | Reusable config, templates, conventions |
| 4 (product) | `output/` | Per-run artifacts |

---

## Stage Sequence

```
stages/
├── 01-auth/          # Authentication & config
├── 02-operations/    # Core OCI operations (instances, compartments, etc.)
└── 03-mcp/           # OpenCode MCP server integration
```

---

## Quick Start

### Buddy Skill (Zoho Vault OneAuth)
When OCI authentication is needed and session token has expired:
1. Load buddy skill: `skill("buddy")` in current session
2. Attempt buddy automation (15s timeout): open Zoho Vault tab + press OneAuth icon + stop
3. On success: continue with OCI operations
4. On failure: instruct user to manually auth via Zoho Vault + OneAuth (no retries)

### Prerequisite: OCI CLI Installation (RPM via curl)

**OCI CLI MUST ALWAYS be installed** before any OCI operations. Use the curl + local RPM install method:

```bash
# Quick install (see references/oci-cli-install-rpm.md for full procedure)
mkdir -p /tmp/rpm-downloads/deps

# Download main packages + all dependencies (18 RPMs total)
# From developer repo:
curl -L -o /tmp/rpm-downloads/python39-oci-cli-3.95.1-1.0.1.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python39-oci-cli-3.95.1-1.0.1.el9.noarch.rpm"
curl -L -o /tmp/rpm-downloads/python39-oci-sdk-2.189.0-1.0.1.el9.aarch64.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python39-oci-sdk-2.189.0-1.0.1.el9.aarch64.rpm"

# Download Python dependencies (18 RPMs from baseos/appstream/developer/EPEL)
# See references/oci-cli-install-rpm.md for complete curl commands

# Install all
rpm -ivh /tmp/rpm-downloads/deps/*.rpm
rpm -ivh /tmp/rpm-downloads/python39-oci-cli-3.95.1-1.0.1.el9.noarch.rpm \
         /tmp/rpm-downloads/python39-oci-sdk-2.189.0-1.0.1.el9.aarch64.rpm

# Verify
oci --version  # Expected: 3.95.1
```

**Automated (PowerShellAI):** `& "/root/.config/opencode/skills/oci/scripts/Ensure-OCICLI.ps1" -Verbose`

---

### Prerequisite: xdg-open Fix for Browser OAuth (Termux/Android)

**MUST RUN ONCE** before using `oci session authenticate`:

```bash
# Load termux skill for authoritative xdg-open fix and environment verification
skill("termux")

# Stage 01: Environment Setup verifies:
# - Real Termux host (not proot/container)
# - xdg-open → termux-open-url link
# - Browser launch capability

# Quick fix (also in termux/references/xdg-open-termux.md):
ln -sf $(which termux-open-url) $(which xdg-open)
xdg-open https://google.com  # Should open Chrome/Firefox/Samsung Internet
```

This enables `oci session authenticate` to auto-launch the browser on Android. **See `termux/references/browser-oauth-termux.md` for complete flow and `termux/references/proot-containers.md` for container limitations.**

---

### Stage 1: Authenticate (Native CLI Only)

**NEVER manually register API keys or copy URLs.** Use native OCI CLI commands:

**Prerequisite (Termux/Android):** Load termux skill and run environment setup:
```bash
skill("termux")
# Stage 01: Environment Setup verifies xdg-open fix and browser capability
```

```bash
# 1. Ensure xdg-open is fixed (run once on Termux/Android) — termux skill handles this
ln -sf $(which termux-open-url) $(which xdg-open)

# 2. Browser OAuth — single command, auto-launches browser, handles callback
oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera

# 3. Verify session token
oci os ns get --auth security_token

# 4. If you want persistent API key auth instead (optional):
#    Generate key pair (once)
openssl genrsa -out /root/.oci/oci_api_key.pem 2048
openssl rsa -pubout -in /root/.oci/oci_api_key.pem -out /root/.oci/oci_api_key_public.pem

#    Register public key (requires active session from step 2)
oci iam api-key upload --key-file /root/.oci/oci_api_key_public.pem

#    Create config with printf (no cat heredoc)
USER_OCID=$(oci iam user list --all --query "data[0].id" --raw-output)
FINGERPRINT=$(openssl rsa -pubout -outform DER -in /root/.oci/oci_api_key.pem | openssl md5 -c | cut -d' ' -f2)
TENANCY_OCID=$(oci iam tenancy get --query "data.id" --raw-output)
printf "[DEFAULT]\nuser=%s\nfingerprint=%s\ntenancy=%s\nregion=us-sanjose-1\nkey_file=/root/.oci/oci_api_key.pem\n" "$USER_OCID" "$FINGERPRINT" "$TENANCY_OCID" > /root/.oci/config
chmod 600 /root/.oci/config

#    Verify API key auth
oci os ns get
```

**Full documentation:** `references/auth-automation.md` + `termux/references/browser-oauth-termux.md`

### Stage 2: Operations
```bash
# Verify auth (API key - no --auth needed)
oci os ns get

# Or session token
oci os ns get --auth security_token

# List instances
oci compute instance list --compartment-id <ocid>
```

### Stage 3: VM Access (see references/vm-ssh-access.md)
```bash
# SSH to VM
ssh opc@<public-ip>

# VS Code remote
code --remote ssh-remote+oci-vm /path/on/vm
```

### Stage 4: MCP
```bash
opencode mcp add oracle-oci-cloud -- uvx oracle.oci-cloud-mcp-server@latest \
  --env OCI_CONFIG_PROFILE=DEFAULT --env FASTMCP_LOG_LEVEL=ERROR
```

---

## Contracts

Each stage defines a contract in its `CONTEXT.md`:
- **Inputs**: What it reads (config, previous stage output)
- **Process**: What it does (commands, scripts)
- **Outputs**: What it writes (artifacts, updated config)

---

## References

- `references/oci-cli-install-rpm.md` - **OCI CLI installation via RPM (curl + local install, no dnf direct)**
- `references/auth-automation.md` - **Fully automated authentication (API key registration, browser OAuth, no manual steps)**
- `references/callback-mechanism.md` - Browser OAuth callback flow (works natively on Termux/Android)
- `references/cli-navigation.md` - **OCI CLI navigation guide: compartments, instances, VNICs, queries, timeouts**
- `references/error-research-methodology.md` - **Error research protocol: capture → classify → research → decide (avoid loops)**
- `references/mcp-config.md` - MCP server configs
- `references/troubleshooting.md` - Error → fix mapping
- `references/persistent-auth.md` - **API key auth (persistent, no expiry)**
- `references/vm-ssh-access.md` - **SSH to VM, VS Code remote, port forwarding**
- `references/reliable-ssh.md` - **REQUIRED: Reliable SSH every time (proven workflow, no retries, serial console fallback)**
- `references/oci-arm-capacity-php.md` - **Ampere VM capacity automation (PHP, cron-based)**
- `references/buddy-zoho-vault-auth.md` - **Buddy automation: Zoho Vault OneAuth (single action, 15s timeout)**

### Termux Skill References (Authoritative for Android/Termux)
- `termux/references/xdg-open-termux.md` - **xdg-open → termux-open-url fix (why, how, verify)**
- `termux/references/browser-oauth-termux.md` - **Complete browser OAuth flow on Android (callback, intents, troubleshooting)**
- `termux/references/proot-containers.md` - **proot-distro, containers in Termux — what works, what doesn't**
- `termux/references/android-intents.md` - **am start, termux-open-url, Android intent mechanics**
- `termux/references/troubleshooting.md` - **Termux error → fix mapping**
- `termux/references/package-management.md` - **pkg/apt in Termux, proot-distro package installation**

## Config References

- `_config/github-tracking.md` - **GitHub Issues tracking for skill improvements: templates, workflow, automation, health metrics**

---

## Conventions

- Stage folders: zero-padded (`01-`, `02-`, `03-`)
- Artifacts: `[topic]-[type].md` in stage `output/`
- Config in `_config/` is reusable across runs
- No spaces in file/folder names

---

## Dependencies

### Termux Skill (Authoritative for Android/Termux Environment)

**Load command:** `skill("termux")` in CURRENT SESSION

**Location:** Global at `/root/.config/opencode/skills/termux/`

**Action:** Execute `skill("termux")` directly — provides authoritative patterns for:
- `xdg-open` → `termux-open-url` fix (prerequisite for browser OAuth)
- Browser OAuth callback mechanism (works natively on Termux/Android)
- proot/container detection (for headless fallback)
- Android intent mechanics (`termux-open-url`, `am start`)

**Required for:** Any `oci session authenticate` on Termux/Android, Zoho Vault OneAuth via Buddy skill.

**References:** `termux/references/xdg-open-termux.md`, `termux/references/browser-oauth-termux.md`, `termux/references/proot-containers.md`, `termux/references/android-intents.md`

---

### Buddy Skill (Zoho Vault OneAuth)

**Load command:** `skill("buddy")` in CURRENT SESSION

**Location:** Global at `/root/.opencode/skill/buddy/` (or workspace copy)
**Action:** Execute `skill("buddy")` directly — do NOT check available skills list first.
**Usage:** 15s timeout automation — open Zoho Vault tab in Ulaa → press OneAuth icon → stop. No further interaction.

**Fallback (genuine load failure only):** If `skill("buddy")` fails with actual error (not "not in list"), instruct user to manually open Ulaa → Zoho Vault → press OneAuth icon.

### Coding Skill (Command Execution Discipline & GitHub Issues)

**Load command:** `skill("coding")` in CURRENT SESSION

**Location:** Global at `/root/.opencode/skill/coding/`
**Action:** Execute `skill("coding")` directly — provides mandatory timeout discipline, process signal discipline, and command execution patterns.
**Enforces:** All `bash` commands MUST include `timeout N` prefix per tier table in coding skill.
**Required for:** Any OCI CLI operations, SSH commands, package installs, network calls.

**GitHub Issues Integration (from coding skill):**
- Use GitHub Issues for tracking OCI tasks, bugs, and feature requests
- Follow issue lifecycle: Open → Triage → Ready → In Progress → In Review → Done
- Use structured issue templates (bug report, feature request, question)
- Link PRs to issues with `closes #123` for auto-closure
- Use GitHub Projects (beta) with custom fields: Status, Priority, Area, Target Release, Size
- Apply label taxonomy: `type:`, `status:`, `priority:`, `area:`, `size:`
- See `_config/github-workflows.md` and `_config/issue-lifecycle.md` in coding skill for full reference

### PowerShellAI Skill (Robust CLI Installation & AI-Assisted Automation)

**Load command:** `skill("powershellai")` in CURRENT SESSION

**Location:** Global at `/root/.config/opencode/skills/powershellai/`
**Action:** Execute `skill("powershellai")` directly — provides mandatory robust automation patterns for CLI installation and long-running operations.
**Enforces:** 
- **OCI CLI MUST ALWAYS be installed** — if missing, immediately invoke PowerShellAI automation script
- **Synchronous execution with live output** — no background jobs, users see every progress line
- **Explicit timeout handling with kill switch** — every external command has timeout + process-tree kill
- **AI-assisted error analysis on EVERY failure** — invoke `ai` to analyze and suggest recovery
- **Session persistence for long operations** — named PSAI sessions with export on failure
- **Streaming AI responses** — users see AI analysis in real-time

**Required for:** OCI CLI auto-installation, any long-running OCI operations (>5 min), error recovery loops.

**Integration Rules (from powershellai skill):**
- Call the PowerShell script directly — never reimplement logic in bash: `& "/absolute/path/Ensure-OCICLI.ps1" -Verbose`
- Use absolute paths — `$env:SKILL_VAR` is NOT reliably set
- Pass timeouts explicitly — don't rely on defaults: `-PackageManagerTimeout 180 -ScriptTimeout 300`
- Capture and act on exit code — 0 = success, 1 = failed: `if ($LASTEXITCODE) { throw "Install failed" }`
- Create GitHub issue on failure — before AND after install attempt: `gh issue create ...` in catch block

**Script Location:** `/root/.config/opencode/skills/oci/scripts/Ensure-OCICLI.ps1` (uses powershellai patterns)

---

## Self-Learning Loop

**Every OCI skill session MUST include a self-learning cycle:**

1. **Capture** — Record exact errors, commands, and context immediately when things go wrong
2. **Classify** — Categorize error: Auth / Capacity / State / Network / Config / Service (≤30 sec)
3. **Research** — Use SearXNG + OCI docs + session history (1-5 min)
4. **Decide** — Apply targeted fix / backoff retry / escalate
5. **Update Skill** — If a new pattern is discovered, load meta skill and update OCI skill references:
   - Add to `references/error-research-methodology.md` (error tables)
   - Add to `references/cli-navigation.md` (new CLI patterns)
   - Add to `references/troubleshooting.md` (error → fix mapping)
   - Update `CONTEXT.md` (current state, VM details)
6. **Create GitHub Issue** — **MANDATORY** for any identified "known issue" or recurring pattern:
   - Create issue in `DiazOlveraCo/oracle-troubleshooting` (or skill repo `DiazOlveraCo/oci-skill` for skill bugs)
   - Use bug report template from `_config/github-tracking.md`
   - Link issue in commit message: `fix(oci): <description> - closes #<issue>`
   - **Do NOT skip this step** — if you document a known issue in the skill, an issue MUST exist

**Anti-Patterns Blocked by This Loop:**
- ❌ Retry same command 5+ times without research
- ❌ Ignore error codes/messages
- ❌ Try random fixes without classification
- ❌ Assume "it worked before" without verification
- ❌ Skip state transition waits (STOPPING → RUNNING)
- ❌ Document "known issue" in skill without creating GitHub issue

**Escalation Path (when self-fix fails):**
1. Serial console connection → direct VM access
2. System logs (`journalctl`, `dmesg`)
3. Cloud-init logs
4. Boot volume detach/attach repair
5. Recreate instance (last resort)

**Self-Learning Trigger:** Any error that takes >2 retries or >10 minutes to resolve MUST trigger a skill update via meta skill **AND** GitHub issue creation.

---

## GitHub Issues Workflow (OCI-Specific)

### Issue Templates for OCI Tasks
Create issues in the OCI troubleshooting repo (`DiazOlveraCo/oracle-troubleshooting`):

**Bug Report Template:**
```markdown
## OCI Bug Report
**Region:** us-sanjose-1
**Resource:** (Instance / VCN / Subnet / etc.)
**OCID:** (if applicable)
**Error:** (exact error message)
**Command:** (command that failed)
**Expected:** (what should happen)
**Actual:** (what happened)
**Timeout Used:** (yes/no, duration)
```

**Feature Request Template:**
```markdown
## OCI Feature Request
**Goal:** (what you want to achieve)
**Current Workflow:** (how you do it now)
**Proposed Solution:** (CLI command, script, automation)
**Priority:** (Critical / High / Medium / Low)
```

### Project Board Setup
Use GitHub Projects (beta) with these custom fields:
- **Status:** Triage → Ready → In Progress → In Review → Done
- **Priority:** Critical → High → Medium → Low
- **Area:** Compute / Network / Storage / Identity / MCP / SSH
- **Target Release:** Sprint 1, Sprint 2, Backlog
- **Size:** 1, 2, 3, 5, 8, 13 (story points)

### Automation
- Built-in workflow: Issue closed → Status = Done
- Built-in workflow: PR opened → Status = In Review
- GitHub Action: Auto-label by file path (`cli-navigation.md` → `area:cli`)
- Stale bot: Archive after 30 days of no activity (exempt `priority:critical`)

---

## Decision Checklist (Must ALL Pass Before Retry)

- [ ] Error captured exactly (command, output, timestamp)
- [ ] Error classified (Auth / Capacity / State / Network / Config / Service)
- [ ] Research performed (SearXNG + OCI docs + session history)
- [ ] Root cause hypothesis formed
- [ ] Targeted fix identified
- [ ] State verified (instance RUNNING, SSH reachable, auth valid)
- [ ] Max retries not exceeded (3 for same error)

If any check fails → **STOP**, research more, update skill, then retry.