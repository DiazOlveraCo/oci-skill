# Stage 02: Core Operations

## Inputs
| Artifact | Location | Format |
|----------|----------|--------|
| Valid auth | Stage 01 output | Session token + config |
| Compartment OCID | User input / discovery | OCID string |
| Region | `~/.oci/config` | String |
| Instance OCID (for SSH) | `CONTEXT.md` or user input | OCID string |

## Process
1. **Load PowerShellAI skill** — `skill("powershellai")` in CURRENT SESSION (required for robust automation)
2. **Verify OCI CLI is installed** (auto-install if missing — see Stage 01, uses PowerShellAI automation)
3. Execute OCI CLI commands for resource management. **MANDATORY: Use reliable-ssh.md workflow for all VM access.**

## Outputs
| Artifact | Location | Format |
|----------|----------|--------|
| Instance list | `output/instances.json` | JSON |
| Compartment list | `output/compartments.json` | JSON |
| AD list | `output/availability-domains.json` | JSON |
| SSH verification | `output/ssh-verify.log` | Log |

## Commands

### Discover Root Compartment
```bash
ROOT_COMP=$(oci iam compartment list --all --compartment-id-in-subtree true \
  --query "data[?id=='compartment-id'].id | [0]" --raw-output --auth security_token)
```

### List All Instances (All Compartments)
```bash
oci compute instance list --compartment-id $ROOT_COMP --all --auth security_token
```

### List Instances in Specific Compartment
```bash
oci compute instance list --compartment-id <compartment_ocid> --auth security_token
```

### List Availability Domains
```bash
oci iam availability-domain list --compartment-id <compartment_ocid> --auth security_token
```

### Get ARM-Compatible Images
```bash
oci compute image list --compartment-id <compartment_ocid> \
  --shape VM.Standard.A1.Flex --operating-system "Canonical Ubuntu" \
  --sort-by TIMECREATED --sort-order DESC --auth security_token
```

### List Subnets in AD
```bash
oci network subnet list --compartment-id <compartment_ocid> \
  --availability-domain <ad_name> --lifecycle-state AVAILABLE --auth security_token
```

### Launch Free Tier Instance (Shape per User)
```bash
# Shape is user-specified — examples:
# AMD: VM.Standard.E2.1.Micro (1 OCPU, 1GB)
# ARM: VM.Standard.A1.Flex (up to 4 OCPU, 24GB)

oci compute instance launch \
  --compartment-id <compartment_ocid> \
  --availability-domain <ad_name> \
  --subnet-id <subnet_ocid> \
  --image-id <image_ocid> \
  --shape "<user-specified-shape>" \
  --shape-config '<user-specified-config>' \
  --display-name "<user-specified-name>-$(date +%s)" \
  --metadata '{"ssh_authorized_keys":"<ssh_pub_key>"}' \
  --wait-for-state RUNNING --max-wait-seconds 600 \
  --auth security_token
```

### Network & Security List Operations (see references/network-security.md)
```bash
# List VCNs
oci network vcn list --compartment-id $ROOT_COMP --auth security_token

# List Security Lists for a VCN
oci network security-list list --compartment-id $ROOT_COMP --vcn-id <vcn_ocid> --auth security_token

# Add SSH (Port 22) Ingress Rule to Security List
oci network security-list update --security-list-id <security_list_ocid> \
  --ingress-security-rules '[{"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"max":22,"min":22}},"description":"Allow SSH from anywhere"}]' \
  --force --auth security_token

# List NSGs
oci network nsg list --compartment-id $ROOT_COMP --vcn-id <vcn_ocid> --auth security_token

# Add SSH Rule to NSG
oci network nsg update --nsg-id <nsg_ocid> \
  --ingress-security-rules '[{"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"max":22,"min":22}},"description":"Allow SSH"}]' \
  --force --auth security_token
```

### Capacity Loop (Free Tier — Shape per User)
```bash
# Shape and config are user-specified
while true; do
  oci compute instance launch \
    --compartment-id $COMP --availability-domain $AD \
    --subnet-id $SUBNET --image-id $IMAGE \
    --shape "$SHAPE" \
    --shape-config "$SHAPE_CONFIG" \
    --display-name "$DISPLAY_PREFIX-$(date +%s)" \
    --metadata '{"ssh_authorized_keys":"'"$SSH_KEY"'"}' \
    --wait-for-state RUNNING --max-wait-seconds 600 \
    --auth security_token && break
  echo "Capacity full, retrying in 60s..."
  sleep 60
done
```

## Reliable SSH Access (MANDATORY — See references/reliable-ssh.md)

**Every VM access MUST follow the exact sequence in `references/reliable-ssh.md`. No exceptions.**

### Quick Reference (Full workflow in reliable-ssh.md)

```bash
# 1. Verify RUNNING state (not STOPPING/STARTING)
timeout 10 oci compute instance get --instance-id $INSTANCE_OCID \
  --query "data.\"lifecycle-state\"" --raw-output --auth security_token
# Must return: RUNNING

# 2. MANDATORY wait after RUNNING
sleep 45

# 3. Test SSH (max 2 retries)
for i in 1 2; do
  timeout 15 ssh -o ConnectTimeout=10 -o BatchMode=yes oci "echo SSH_OK" && break
  sleep 10
done

# 4. If SSH fails after 2 retries → SERIAL CONSOLE (guaranteed)
# One-time setup:
timeout 30 oci compute instance-console-connection create \
  --instance-id $INSTANCE_OCID \
  --ssh-public-key-file ~/.ssh/oci_vm_key.pub \
  --auth security_token

# Connect:
CONN_ID=$(timeout 15 oci compute instance-console-connection list --instance-id $INSTANCE_OCID --query "data[0].id" --raw-output --auth security_token)
timeout 30 ssh -i ~/.ssh/oci_vm_key \
  -o ProxyCommand="ssh -W %h:%p -p 443 $CONN_ID@instance-console.us-sanjose-1.oci.oraclecloud.com" \
  -o ConnectTimeout=20 \
  $INSTANCE_OCID

# 5. In serial console, fix permanently:
sudo systemctl enable --now sshd
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
sudo dnf install -y haveged && sudo systemctl enable --now haveged
sudo systemctl restart oracle-cloud-agent
exit

# 6. Verify SSH works
timeout 10 ssh oci "echo SSH_OK && hostname && uptime"
```

### State Management (Fix State Before SSH)

| Current State | Action | Wait For | Then |
|---------------|--------|----------|------|
| STOPPING (>5 min) | `oci compute instance action --action STOP` | STOPPED | START → RUNNING → wait 45s |
| STOPPED | `oci compute instance action --action START` | RUNNING | wait 45s |
| RUNNING (SSH fails) | `oci compute instance action --action SOFTRESET` | RUNNING | wait 45s |
| TERMINATED | Launch new instance with same shape + SSH key | RUNNING | wait 45s |

### SSH Config (Must Match Exactly)

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

**Critical:** `IdentityFile` must be the exact key authorized on the VM (`~/.ssh/oci_vm_key` copied from `ssh-key-2026-10-31(3).key` — the 4th key in Downloads matching instance launch time).

## Validation

```bash
# Check instance state
oci compute instance list --compartment-id $COMP --auth security_token \
  --query "data[*].{name:\"display-name\", state:\"lifecycle-state\", id:id, shape:\"shape\", ad:\"availability-domain\"}"

# Verify SSH health (after connection)
ssh oci "systemctl is-active sshd && systemctl is-enabled sshd && getenforce && systemctl is-active haveged && systemctl is-active oracle-cloud-agent"
# All must return: active, active, Permissive, active, active
```

## Error Handling (Per error-research-methodology.md)

| Error | Classification | Action |
|-------|----------------|--------|
| Instance STOPPING >5 min | State | Force STOP → START |
| SSH "Connection refused" | Config | Serial console → fix sshd |
| SSH "timeout during banner" | State/Config | Wait 45s after RUNNING, then serial console |
| "Permission denied (publickey)" | Config | Verify `IdentityFile` matches authorized key |
| "Out of host capacity" | Capacity | Use capacity script (oci-arm-capacity-php.md) or adjust shape/config |

## Required Timeouts (Per coding skill)

| Operation | Timeout |
|-----------|---------|
| Auth verification | `timeout 10` |
| Instance list | `timeout 15` |
| Instance action (START/STOP/RESET) | `timeout 30` |
| State polling | `timeout 15` per poll, max 180s total |
| SSH test | `timeout 15` |
| Serial console connect | `timeout 60` |
| Console connection create | `timeout 30` |