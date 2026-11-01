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

### Stage 1: Authenticate
```bash
# Browser OAuth (Termux/Android works natively)
oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera

# Or API keys for headless
oci setup config
```

### Stage 2: Operations
```bash
# Verify auth
oci os ns get --auth security_token

# List instances
oci compute instance list --compartment-id <ocid> --auth security_token
```

### Stage 3: MCP
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

---

## Conventions

- Stage folders: zero-padded (`01-`, `02-`, `03-`)
- Artifacts: `[topic]-[type].md` in stage `output/`
- Config in `_config/` is reusable across runs
- No spaces in file/folder names