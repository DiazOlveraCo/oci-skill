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
| SSH | `ssh oci` or `ssh oci-micro` (Oracle Linux 9, user: opc) |

## Instance State Management (CLI)

### Check Instance State
```bash
timeout 10 oci compute instance get --instance-id <ocid> --query "data.\"lifecycle-state\"" --raw-output --auth security_token
```

### Stop Stuck Instance (STOPPING → STOPPED)
```bash
oci compute instance action --instance-id <ocid> --action STOP --auth security_token
# Wait for STOPPED state
oci compute instance get --instance-id <ocid> --query "data.\"lifecycle-state\"" --raw-output --auth security_token
```

### Start Instance (STOPPED → RUNNING)
```bash
oci compute instance action --instance-id <ocid> --action START --auth security_token
# Wait for RUNNING state
oci compute instance get --instance-id <ocid> --query "data.\"lifecycle-state\"" --raw-output --auth security_token
```

### Restart Instance (RUNNING → STOPPING → RUNNING)
```bash
oci compute instance action --instance-id <ocid> --action SOFTRESET --auth security_token
# Or HARDRESET for force
oci compute instance action --instance-id <ocid> --action RESET --auth security_token
```

### Force Stop if STUCK in STOPPING
```bash
# If instance stuck in STOPPING for >5 min
oci compute instance action --instance-id <ocid> --action STOP --auth security_token
# Then START
oci compute instance action --instance-id <ocid> --action START --auth security_token
```

## Serial Console Workflow (Reliable SSH Fix)

### Create Console Connection (One-time)
```bash
oci compute instance-console-connection create \
  --instance-id <instance-ocid> \
  --ssh-public-key-file ~/.ssh/oci_vm_key.pub \
  --auth security_token \
  --query "data.id" --raw-output
```

### Connect via Serial Console
```bash
# Get connection details
CONN_ID=$(oci compute instance-console-connection list --instance-id <instance-ocid> --query "data[0].id" --raw-output --auth security_token)
CONN_STRING=$(oci compute instance-console-connection get --instance-console-connection-id $CONN_ID --query "data.connection-string" --raw-output --auth security_token)

# Connect (example format):
ssh -i ~/.ssh/oci_vm_key -o ProxyCommand="ssh -W %h:%p -p 443 $CONN_ID@instance-console.us-sanjose-1.oci.oraclecloud.com" <instance-ocid>
```

### Fix SSH Daemon via Serial Console
Once connected (you'll be at a login prompt or root shell):
```bash
# Check SSH status
systemctl status sshd

# Start and enable SSH
systemctl start sshd
systemctl enable sshd

# Fix SELinux (Oracle Linux 9 common issue)
getenforce
# If "Enforcing":
setenforce 0
# Permanent fix:
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Check for oracle-cloud-agent conflict
systemctl status oracle-cloud-agent
# If it's blocking SSH, restart it:
systemctl restart oracle-cloud-agent

# Install haveged for entropy (micro instances)
dnf install -y haveged
systemctl enable --now haveged
```

## OCI Mobile App Workflow (Buddy/Phone Automation)

### Restart Instance via OCI Mobile App
1. **Open app**: `com.oracle.oci.console.ocimobile`
2. **Tap "Compute"** (element with desc="Compute")
3. **Tap your instance** (e.g., "instance-20261030-1802")
4. **Tap "Restart"** button
5. **Confirm** restart in dialog
6. **Wait** for status to show "Stopping" → "Running"
7. **Wait 30-60s** after RUNNING before SSH

### Force Stop if Stuck in STOPPING
1. Navigate to instance details
2. If status shows "Stopping" for >5 min:
   - Tap "Stop" (if enabled) or wait
   - If still stuck, use CLI: `oci compute instance action --action STOP --instance-id <ocid> --wait-for-state STOPPED`
   - Then START: `oci compute instance action --action START --instance-id <ocid> --wait-for-state RUNNING`

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| "NotAuthenticated" 401 | Token expired | Re-run `oci session authenticate` |
| "Missing option --compartment-id" | Need compartment OCID | Use tenancy OCID as root compartment |
| Empty instance list | Wrong compartment | Try tenancy root OCID |
| "Config file invalid: user missing" | Config corrupted by auth | Add `user=ocid1.user.oc1..aaaa...` to config |
| Command hangs | Network/timeout | Add `timeout N` prefix |
| "Out of host capacity" | No free tier capacity | Retry via PHP script or wait |
| SSH "Connection refused" | sshd not running | See `references/vm-ssh-access.md` — OCI SSH Daemon Issues |
| SSH "Connection timed out during banner exchange" | SSH daemon hung/starting | Wait 30-60s after RUNNING, use serial console |
| Instance stuck in STOPPING | OCI agent issue | `oci compute instance action --action STOP` then `START` |

## Notes

- All commands require `--auth security_token` for session token auth
- Use `--auth api_key` if using API key config (no `security_token_file`)
- Always add `timeout` to prevent hanging
- Tenancy OCID works as root compartment for most operations