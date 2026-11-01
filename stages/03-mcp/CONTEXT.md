# Stage 03: MCP Integration

## Inputs
| Artifact | Location | Format |
|----------|----------|--------|
| Valid OCI config | Stage 01 output | `~/.oci/config` |
| Profile name | Config | String (DEFAULT) |

## Process
Configure OpenCode MCP server to use Oracle OCI Cloud MCP.

## Outputs
| Artifact | Location | Format |
|----------|----------|--------|
| OpenCode config | `~/.config/opencode/opencode.json` | JSON |
| MCP server registered | `opencode mcp list` | Status |

## Commands

### Add MCP Server via CLI
```bash
opencode mcp add oracle-oci-cloud -- uvx oracle.oci-cloud-mcp-server@latest \
  --env OCI_CONFIG_PROFILE=DEFAULT \
  --env FASTMCP_LOG_LEVEL=ERROR
```

### Manual Config (~/.config/opencode/opencode.json)
```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "oracle-oci-cloud": {
        "type": "local",
        "command": ["uvx", "oracle.oci-cloud-mcp-server@latest"],
        "environment": {
          "OCI_CONFIG_PROFILE": "DEFAULT",
          "FASTMCP_LOG_LEVEL": "ERROR"
        }
      }
    }
  }
}
```

### Verify Connection
```bash
opencode mcp list
# ✓ oracle-oci-cloud  connected
```

## Alternative MCP Servers

| Server | Package | Use Case |
|--------|---------|----------|
| oci-cloud-mcp-server | `oracle.oci-cloud-mcp-server@latest` | Official, Python SDK, recommended |
| oci-api-mcp-server | `oracle.oci-api-mcp-server@latest` | CLI-backed, for CLI-specific commands |
| oci-javascript-mcp-server | npm | Node.js variant |

## HTTP Mode (Remote)
```bash
# On server
ORACLE_MCP_HOST=0.0.0.0 ORACLE_MCP_PORT=8888 \
  OCI_REGION=us-sanjose-1 \
  IDCS_DOMAIN=<domain> IDCS_CLIENT_ID=<id> \
  IDCS_CLIENT_SECRET=<secret> IDCS_AUDIENCE=<audience> \
  uvx oracle.oci-cloud-mcp-server@latest

# Client config
{
  "mcpServers": {
    "oracle-oci-cloud": {
      "type": "streamableHttp",
      "url": "http://<server-ip>:8888/mcp"
    }
  }
}
```

## Validation
```bash
opencode mcp list
# Should show connected status
```