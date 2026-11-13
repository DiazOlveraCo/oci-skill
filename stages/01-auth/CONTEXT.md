# Stage 01: Authentication

## Inputs
| Artifact | Location | Format |
|----------|----------|--------|
| Tenancy name | User input | String |
| Region | User input | String (e.g., us-sanjose-1) |
| Existing config | `~/.oci/config` | INI |

## Process
1. **MANDATORY: Load PowerShellAI skill** — `skill("powershellai")` in CURRENT SESSION (required for robust automation)
2. **MANDATORY: Verify OCI CLI is installed — if missing, create GitHub issue AND install immediately via PowerShellAI automation**
3. Run browser OAuth flow
4. Create session token + keys in `~/.oci/sessions/DEFAULT/`
5. Update `~/.oci/config` with profile

## Outputs
| Artifact | Location | Format |
|----------|----------|--------|
| Session token | `~/.oci/sessions/DEFAULT/token` | JWT |
| Private key | `~/.oci/sessions/DEFAULT/oci_api_key.pem` | PEM |
| Public key | `~/.oci/sessions/DEFAULT/oci_api_key_public.pem` | PEM |
| Updated config | `~/.oci/config` | INI |

## OCI CLI Verification (MANDATORY — Run FIRST, Before ANYTHING Else) — PowerShell

**PREREQUISITE: Load PowerShellAI skill first** — `skill("powershellai")` — required for AI-assisted error analysis and robust automation patterns.

**CORE RULE: OCI CLI must be verified and working before ANY OCI command. If not installed, the agent MUST:**
1. **Create a GitHub issue** in `DiazOlveraCo/oci-skill` with title: `fix(oci): OCI CLI missing — auto-install triggered`
2. **Install immediately** using the PowerShellAI automation script (follows robust automation patterns: sync execution, timeouts, AI error analysis, session persistence)
3. **Verify installation succeeds** before proceeding

```powershell
# STEP 1: Check if OCI CLI exists
if (-not (Get-Command oci -ErrorAction SilentlyContinue)) {
    Write-Error "OCI CLI NOT FOUND — Creating GitHub issue and installing immediately"
    
    # STEP 2: Create GitHub issue (using gh CLI)
    $issueBody = @"
## Bug Report: OCI CLI Missing
**Skill:** oci
**File:** stages/01-auth/CONTEXT.md
**Error:** OCI CLI not installed — agent cannot execute any OCI commands
**Context:** OCI skill loaded but CLI missing from PATH
**Expected:** OCI CLI should be pre-installed or auto-installed on skill load
**Actual:** CLI missing, operation blocked
**Auto-Fix:** Running Ensure-OCICLI.ps1 PowerShellAI installer
"@
    
    gh issue create --repo DiazOlveraCo/oci-skill \
        --title "fix(oci): OCI CLI missing — auto-install triggered" \
        --body $issueBody \
        --label "type:bug,area:cli,priority:critical,skill:oci"
    
    # STEP 3: Install via PowerShellAI automation (ABSOLUTE PATH REQUIRED)
    & "/root/.config/opencode/skills/oci/scripts/Ensure-OCICLI.ps1" -Verbose
    
    # STEP 4: Verify installation
    if (-not (Get-Command oci -ErrorAction SilentlyContinue)) {
        throw "FATAL: OCI CLI installation failed after GitHub issue creation"
    }
    Write-Host "OCI CLI installed and verified: $(oci --version)" -ForegroundColor Green
}
else {
    Write-Host "OCI CLI verified: $(oci --version)" -ForegroundColor Green
}
```

**The script (`scripts/Ensure-OCICLI.ps1`) handles:**
- Platform detection: dnf, apt, pacman, apk, yum, zypper
- Official Oracle installer fallback (with auto-cleanup of existing /root/lib/oracle-cli)
- Post-install verification
- PSAI-powered AI error analysis on failure
- Verbose logging and force reinstall options

> **⚠️ CRITICAL**: Past sessions proved agents IGNORED this script and ran broken Oracle installer workarounds. The `$env:OPENCODE_SKILL_OCI` variable is NOT set. USE ABSOLUTE PATH. Do NOT skip GitHub issue creation.

## Commands
```bash
# FRESH AUTH — Wipe expired sessions, then authenticate
rm -rf /root/.oci/sessions/DEFAULT/token /root/.oci/sessions/DEFAULT/oci_session_token_* 2>/dev/null
oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera
# Enter profile name when prompted (e.g., DEFAULT)

# Refresh expired token (only if fresh auth failed)
oci session authenticate --profile-name DEFAULT --region us-sanjose-1 --auth security_token

# API key setup (headless)
oci setup config
```

## Fresh Auth Verification (MANDATORY — Run AFTER auth)
```bash
oci os ns get --auth security_token
# Returns namespace = success
```

## Termux/Android Fix (Run ONCE on Real Host)
```bash
pkg install -y termux-api
ln -sf $(which termux-open) $(which xdg-open)
```

## Anti-Patterns (Agent MUST Avoid)

| Anti-Pattern | Correct Behavior |
|--------------|------------------|
| ❌ Agent prints URL and tells user "copy this and open in browser" | ✅ Agent runs command directly; browser auto-launches or URL prints |
| ❌ Agent runs `oci session authenticate` inside proot-distro and waits | ✅ Agent runs command directly; if URL prints, user opens it manually |
| ❌ Agent says "if browser doesn't open, copy the URL manually" | ✅ Agent NEVER suggests manual URL copy; just runs the command |
| ❌ Agent retries same auth command multiple times | ✅ Agent runs fresh auth ONCE |
| ❌ Agent creates shell scripts to automate browser launch | ✅ Agent runs command directly in interactive terminal |