# OCI CLI Navigation Guide

## Overview

This reference covers how to navigate Oracle Cloud Infrastructure using the OCI CLI. The CLI is installed at `/root/lib/oracle-cli/bin/oci` and uses session token auth (browser OAuth) or API key auth.

## Prerequisites

### Auth Verification (with timeout)
```bash
# Always use --auth security_token for session token auth
# Always add timeout to prevent hanging
export SUPPRESS_LABEL_WARNING=True
timeout 10 oci os ns get --auth security_token
# Returns namespace = success
```

### Environment Variables
```bash
export SUPPRESS_LABEL_WARNING=True
export OCI_CONFIG_PROFILE=DEFAULT
export OCI_CLI_AUTH=security_token
```

## Core Navigation Patterns

### 1. Tenancy & Compartment Discovery
```bash
# Get tenancy OCID (root compartment)
TENANCY=$(oci iam tenancy get --query "data.id" --raw-output --auth security_token)

# List all compartments (including nested)
oci iam compartment list --all --compartment-id-in-subtree true --auth security_token

# Get root compartment (first result)
ROOT_COMP=$(oci iam compartment list --all --compartment-id-in-subtree true --query "data[0].id" --raw-output --auth security_token)
```

### 2. Compute Instances
```bash
# List all instances in tenancy
oci compute instance list --compartment-id "$TENANCY" --all --auth security_token

# List with specific fields
oci compute instance list --compartment-id "$TENANCY" --all \
  --query "data[*].{name:\"display-name\", state:\"lifecycle-state\", shape:shape, public_ip:\"public-ip\", private_ip:\"private-ip\"}" \
  --auth security_token

# Get instance details
oci compute instance get --instance-id <instance-ocid> --auth security_token

# List VNIC attachments for an instance
oci compute vnic-attachment list --compartment-id "$TENANCY" --instance-id <instance-ocid> --auth security_token

# Get VNIC details (public/private IP)
oci network vnic get --vnic-id <vnic-ocid> --auth security_token
```

### 3. Availability Domains
```bash
# List ADs in region
oci iam availability-domain list --compartment-id "$TENANCY" --auth security_token
```

### 4. Images
```bash
# List images for a shape
oci compute image list --compartment-id "$TENANCY" --shape "VM.Standard.A1.Flex" --auth security_token

# List all images
oci compute image list --compartment-id "$TENANCY" --all --auth security_token
```

### 5. Networking (VCN, Subnets)
```bash
# List VCNs
oci network vcn list --compartment-id "$TENANCY" --auth security_token

# List subnets
oci network subnet list --compartment-id "$TENANCY" --auth security_token

# Get subnet details
oci network subnet get --subnet-id <subnet-ocid> --auth security_token
```

### 6. Block Volumes / Boot Volumes
```bash
# List boot volumes
oci bv boot-volume list --compartment-id "$TENANCY" --availability-domain <ad-name> --auth security_token

# List block volumes
oci bv volume list --compartment-id "$TENANCY" --availability-domain <ad-name> --auth security_token
```

## Common Query Patterns

### JMESPath Queries
```bash
# Filter by state
--query "data[?\"lifecycle-state\"=='RUNNING']"

# Select specific fields
--query "data[*].{name:\"display-name\", ip:\"public-ip\"}"

# Get first item
--query "data[0].id" --raw-output

# Count items
--query "length(data)"
```

### Output Formats
```bash
# Table (human-readable)
--output table

# JSON (default, machine-readable)
--output json

# Raw (single value)
--raw-output
```

## Timeout Discipline

**Always use timeouts** to prevent hanging on network issues:

```bash
# Short operations (auth, namespace)
timeout 10 oci os ns get --auth security_token

# Medium operations (list instances, compartments)
timeout 15 oci compute instance list --compartment-id "$TENANCY" --all --auth security_token

# Long operations (image list, all resources)
timeout 30 oci compute image list --compartment-id "$TENANCY" --all --auth security_token
```

## Auth Refresh

Session tokens expire (~60 min). When commands return 401:

```bash
# Re-authenticate (browser OAuth, callback works on Termux/Android)
oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera --profile-name DEFAULT
# Enter profile name when prompted: DEFAULT

# Or use --no-browser if needed (requires existing config)
oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera --no-browser
```

## Quick Reference: Current VM

| Property | Value |
|----------|-------|
| Name | instance-20261030-1802 |
| Shape | VM.Standard.E2.1.Micro (AMD) |
| State | RUNNING |
| AD | US-SANJOSE-1-AD-1 |
| Public IP | 146.235.239.203 |
| Private IP | 10.1.21.187 |
| SSH | `ssh -i ~/.ssh/oci_vm_key ubuntu@146.235.239.203` |

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| "NotAuthenticated" 401 | Token expired | Re-run `oci session authenticate` |
| "Missing option --compartment-id" | Need compartment OCID | Use tenancy OCID as root compartment |
| Empty instance list | Wrong compartment | Try tenancy root OCID |
| "Config file invalid: user missing" | Config corrupted by auth | Add `user=ocid1.user.oc1..aaaa...` to config |
| Command hangs | Network/timeout | Add `timeout N` prefix |
| "Out of host capacity" | No free tier capacity | Retry via PHP script or wait |

## Notes

- All commands require `--auth security_token` for session token auth
- Use `--auth api_key` if using API key config (no `security_token_file`)
- Always add `timeout` to prevent hanging
- Tenancy OCID works as root compartment for most operations