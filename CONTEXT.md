# OCI Workspace Context

## Identity
- **Provider**: Oracle Cloud Infrastructure (OCI)
- **Region**: us-sanjose-1 (primary)
- **Tenancy**: diazolvera
- **Profile**: DEFAULT

## Current State
- Session token auth configured at `~/.oci/sessions/DEFAULT/`
- Token expires ~60 min, refresh with `oci session authenticate --auth security_token`
- API keys also available for headless persistence (`~/.oci/oci_api_key_pkcs1.pem`)
- Buddy MCP integration available for Zoho Vault OneAuth automation

## Buddy Integration
- Load buddy skill in current OpenCode session for phone automation
- Buddy MCP server: `172.21.154.90:8765` (USB tether) or `127.0.0.1:8765` (local)
- Zoho Vault OneAuth: single action (open tab + press icon + stop), 15s timeout
- On buddy error: fall back to manual instructions (no retries)

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
| Verify auth | `oci os ns get --auth security_token` |
| List compartments | `oci iam compartment list --all --compartment-id-in-subtree true --auth security_token` |
| List instances | `oci compute instance list --compartment-id <ocid> --auth security_token` |
| List ADs | `oci iam availability-domain list --compartment-id <ocid> --auth security_token` |
| Get images | `oci compute image list --compartment-id <ocid> --shape VM.Standard.A1.Flex --auth security_token` |
| Get subnets | `oci network subnet list --compartment-id <ocid> --auth security_token` |

## Environment
```bash
export OCI_CLI_SUPPRESS_FILE_PERMISSIONS_WARNING=True
export OCI_CONFIG_PROFILE=DEFAULT
export OCI_CLI_AUTH=security_token
```