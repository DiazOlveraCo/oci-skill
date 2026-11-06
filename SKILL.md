---
name: oci
description: Work with Oracle Cloud Infrastructure using CLI, MCP, and API operations following ICM (Interpretable Context Methodology)
---

# OCI Skill for OpenCode

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

### Stage 1: Authenticate
```bash
# Browser OAuth (Termux/Android works natively) - expires ~60 min
oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera

# API keys for persistent headless access (recommended for automation)
# One-time setup - see references/persistent-auth.md
cat > ~/.oci/config << 'EOF'
[DEFAULT]
user=ocid1.user.oc1..aaaaaaaake25vifglqspfashsmyionbxqt2tc65tbpd2a3qiixiv7exaskxa
fingerprint=d8:9d:a1:15:8a:05:61:ad:5b:4b:2a:44:a4:be:03:9d
tenancy=ocid1.tenancy.oc1..aaaaaaaasimpzm2iolrderse5ostt5l5djgsdt3slzmmeqzfbluv3ruqmugq
region=us-sanjose-1
key_file=/root/.oci/oci_api_key.pem
EOF
# Register public key in Console → Profile → API Keys first!
```

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

- `references/auth-methods.md` - All authentication methods
- `references/cli-patterns.md` - Common CLI patterns
- `references/mcp-config.md` - MCP server configs
- `references/troubleshooting.md` - Error → fix mapping
- `references/callback-mechanism.md` - Browser OAuth callback flow
- `references/xdg-open-termux.md` - Termux xdg-open fix for browser auth
- `references/persistent-auth.md` - **API key auth (persistent, no expiry)**
- `references/vm-ssh-access.md` - **SSH to VM, VS Code remote, port forwarding**
- `references/oci-arm-capacity-php.md` - **Ampere VM capacity automation (PHP, cron-based)**
- `references/buddy-zoho-vault-auth.md` - **Buddy automation: Zoho Vault OneAuth (single action, 15s timeout)**

---

## Conventions

- Stage folders: zero-padded (`01-`, `02-`, `03-`)
- Artifacts: `[topic]-[type].md` in stage `output/`
- Config in `_config/` is reusable across runs
- No spaces in file/folder names