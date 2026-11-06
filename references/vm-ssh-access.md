# VM SSH Access from OpenCode

## Prerequisites
- Persistent API key auth configured (see `persistent-auth.md`)
- SSH key pair for VM access
- VM running in OCI

## Get VM Details

### List All Instances
```bash
# Get root compartment
ROOT_COMP=$(oci iam compartment list --all --compartment-id-in-subtree true \
  --query "data[?id=='compartment-id'].id | [0]" --raw-output)

# List instances with details
oci compute instance list --compartment-id $ROOT_COMP --all \
  --query "data[*].{name:\"display-name\", id:id, state:\"lifecycle-state\", \
    shape:\"shape\", ad:\"availability-domain\", public_ip:\"public-ip\", \
    private_ip:\"private-ip\"}" --output table
```

### Get Specific Instance
```bash
oci compute instance get --instance-id <instance-ocid> \
  --query "data.{name:\"display-name\", id:id, state:\"lifecycle-state\", \
    public_ip:\"public-ip\", private_ip:\"private-ip\", shape:\"shape\"}"
```

## SSH Access

### From This Machine (if VM has public IP)
```bash
# Ensure SSH key permissions
chmod 600 ~/.ssh/id_rsa

# SSH to VM
ssh opc@<public-ip>
# Or ubuntu@<public-ip> depending on image
```

### SSH Config (~/.ssh/config)
```ssh
Host oci oci-micro
    HostName 146.235.239.203
    User opc
    IdentityFile ~/.ssh/oci_vm_key
    IdentitiesOnly yes
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel ERROR
    ConnectTimeout 20
```

**Usage:**
- `ssh oci` — connect to the OCI VM
- `ssh oci-micro` — same VM (alias for the micro instance)

**OS:** Oracle Linux 9 (user: `opc`)
**Shape:** VM.Standard.E2.1.Micro (AMD, 1 OCPU, 1 GB RAM)

Then: `ssh oci-vm`

### Port Forwarding (for local services on VM)
```bash
# Forward local port 8080 to VM's port 8080
ssh -L 8080:localhost:8080 oci-vm

# Forward multiple ports
ssh -L 8080:localhost:8080 -L 3000:localhost:3000 oci-vm
```

## VS Code Remote Development

### Install Remote SSH Extension
```bash
code --install-extension ms-vscode-remote.remote-ssh
```

### Configure VS Code SSH
1. `Ctrl+Shift+P` → "Remote-SSH: Connect to Host"
2. Add `oci-vm` from config
3. Opens VS Code on remote VM

### Or via CLI
```bash
code --remote ssh-remote+oci-vm /path/on/vm
```

## OpenCode on VM

### Run OpenCode on VM via SSH
```bash
# From local machine
ssh oci-vm "opencode"
```

### Mount VM Files Locally (sshfs)
```bash
# Install sshfs
apt-get install sshfs

# Mount
mkdir -p ~/oci-vm-mount
sshfs oci-vm:/home/opc ~/oci-vm-mount

# Open in local editor
code ~/oci-vm-mount
```

## Automated VM Discovery Script
```bash
#!/bin/bash
# find-vm.sh - Find and SSH to VM by name pattern

PATTERN="${1:-arm-free}"
ROOT_COMP=$(oci iam compartment list --all --compartment-id-in-subtree true \
  --query "data[?id=='compartment-id'].id | [0]" --raw-output)

INSTANCE=$(oci compute instance list --compartment-id $ROOT_COMP --all \
  --query "data[?contains(\"display-name\", '$PATTERN') && \"lifecycle-state\"=='RUNNING'] | [0].{ip:\"public-ip\", id:id}" --raw-output)

if [ -n "$INSTANCE" ]; then
    IP=$(echo $INSTANCE | cut -d' ' -f1)
    echo "Connecting to $IP..."
    ssh opc@$IP
else
    echo "No running VM matching '$PATTERN' found"
fi
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `Permission denied (publickey)` | Check `~/.ssh/id_rsa.pub` in VM's `~/.ssh/authorized_keys` |
| `Connection timeout` | Check security list allows port 22 from your IP |
| `No public IP` | Use bastion host or Cloud Shell SSH |
| `Host key verification failed` | Add `StrictHostKeyChecking no` to SSH config |
| `Connection refused` | **sshd not running** — see "OCI SSH Daemon Issues" below |
| `Connection timed out during banner exchange` | SSH daemon hung or starting — wait 30-60s after reboot, try serial console |

### OCI SSH Daemon Issues (Known Problem)

**Problem**: On Oracle Cloud free tier instances (especially VM.Standard.E2.1.Micro), the `sshd.service` frequently fails to start on boot. This is a documented known issue.

**Symptoms**:
- `Connection refused` immediately after instance shows RUNNING
- `Connection timed out during banner exchange` after initial connection
- SSH works briefly then stops working

**Root Causes**:
1. `sshd.service` fails to start on boot (known OCI issue)
2. SELinux policy blocks SSH on Oracle Linux 9
3. `oracle-cloud-agent` conflicts with SSH startup
4. Insufficient entropy for SSH key generation on micro instances

**Workarounds**:

1. **Wait after reboot**: After instance shows RUNNING, wait 30-60 seconds before SSH
2. **Serial Console** (reliable access):
   ```bash
   # Create console connection
   oci compute instance-console-connection create --instance-id <ocid> --ssh-public-key-file ~/.ssh/key.pub
   
   # Connect via console
   ssh -i ~/.ssh/key -o ProxyCommand="ssh -W %h:%p -p 443 <connection-id>@instance-console.<region>.oci.oraclecloud.com" <instance-ocid>
   ```
3. **Fix via serial console**:
   ```bash
   # Once in serial console (as root):
   systemctl status sshd
   systemctl start sshd
   systemctl enable sshd
   
   # Check SELinux
   getenforce
   # If Enforcing, temporarily allow:
   setenforce 0
   # Or fix permanently:
   sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
   ```
4. **Boot volume repair** (last resort):
   - Stop instance
   - Detach boot volume
   - Attach to another instance
   - Fix `/etc/ssh/sshd_config`, `/etc/selinux/config`
   - Reattach to original instance

### OCI Free Tier Specific Issues

| Shape | Issue | Mitigation |
|-------|-------|------------|
| `VM.Standard.E2.1.Micro` (AMD) | Low entropy → SSH key gen fails | `sudo dnf install haveged && sudo systemctl enable haveged` |
| `VM.Standard.A1.Flex` (ARM) | Capacity limits, different OS images | Use Oracle Linux 8 instead of 9 |
| All free tier | Instance may be stopped by OCI if "idle" | Set up cron to ping instance, or use monitoring |

### Reliable SSH Connection Script
```bash
#!/bin/bash
# ssh-oci-reliable.sh - Connect with retries and console fallback

HOST="oci"  # or oci-micro
MAX_RETRIES=5
RETRY_DELAY=15

for i in $(seq 1 $MAX_RETRIES); do
    if ssh -o ConnectTimeout=10 -o BatchMode=yes $HOST "echo OK" 2>/dev/null; then
        echo "SSH OK on attempt $i"
        exec ssh $HOST "$@"
    fi
    echo "Attempt $i failed, waiting ${RETRY_DELAY}s..."
    sleep $RETRY_DELAY
done

echo "All SSH attempts failed. Trying serial console..."
# Fallback: use OCI CLI to create console connection
# (requires console connection already created)
```

## Current VM Targets
Run `oci compute instance list --all --query "data[?\"lifecycle-state\"=='RUNNING']"` to see running VMs.