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

## Serial Console Workflow (Reliable SSH Fix)

### Create Console Connection (One-time)
```bash
oci compute instance-console-connection create \
  --instance-id <instance-ocid> \
  --ssh-public-key-file ~/.ssh/oci_vm_key.pub \
  --query "data.id" --raw-output
```

### Connect via Serial Console
```bash
# Get connection details
CONN_ID=$(oci compute instance-console-connection list --instance-id <instance-ocid> --query "data[0].id" --raw-output)
CONN_STRING=$(oci compute instance-console-connection get --instance-console-connection-id $CONN_ID --query "data.connection-string" --raw-output)

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

## Complete Installation Workflow (PHP Capacity Automation Script)

### Prerequisites on VM
```bash
# 1. Fix SSH daemon (run via serial console if needed)
sudo systemctl enable --now sshd
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
sudo dnf install -y haveged && sudo systemctl enable --now haveged

# 2. Install PHP and Composer
sudo dnf install -y php php-cli php-mbstring php-xml php-curl php-zip unzip git
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php --install-dir=/usr/local/bin --filename=composer
php -r "unlink('composer-setup.php');"
```

### Deploy Capacity Script
```bash
# 1. Clone the repo
cd /home/opc
rm -rf oci-arm-host-capacity
git clone https://github.com/hitrov/oci-arm-host-capacity.git
cd oci-arm-host-capacity

# 2. Install dependencies
composer install --no-dev --optimize-autoloader

# 3. Configure .env
cp .env.example .env
# Edit .env with your OCI credentials:
# - TENANCY_OCID
# - USER_OCID
# - FINGERPRINT
# - PRIVATE_KEY_PATH
# - REGION (us-sanjose-1)
# - COMPARTMENT_OCID
# - SHAPE (VM.Standard.A1.Flex)
# - OCPU_COUNT, MEMORY_IN_GBS

# 4. Test run
php src/capacity.php

# 5. Add cron for periodic retries (every 5 minutes)
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/bin/php /home/opc/oci-arm-host-capacity/src/capacity.php >> /home/opc/capacity.log 2>&1") | crontab -

# 6. Monitor
tail -f /home/opc/capacity.log
```

### .env Template for us-sanjose-1 Ampere A1
```env
TENANCY_OCID=ocid1.tenancy.oc1..aaaaaaaasimpzm2iolrderse5ostt5l5djgsdt3slzmmeqzfbluv3ruqmugq
USER_OCID=ocid1.user.oc1..aaaaaaaake25vifglqspfashsmyionbxqt2tc65tbpd2a3qiixiv7exaskxa
FINGERPRINT=d8:9d:a1:15:8a:05:61:ad:5b:4b:2a:44:a4:be:03:9d
PRIVATE_KEY_PATH=/home/opc/.oci/oci_api_key.pem
REGION=us-sanjose-1
COMPARTMENT_OCID=ocid1.compartment.oc1..aaaaaaaake25vifglqspfashsmyionbxqt2tc65tbpd2a3qiixiv7exaskxa
SHAPE=VM.Standard.A1.Flex
OCPU_COUNT=4
MEMORY_IN_GBS=24
AVAILABILITY_DOMAIN=aXzQ:US-SANJOSE-1-AD-1
IMAGE_OCID=ocid1.image.oc1.us-sanjose-1.aaaaaaaaxxxxx  # Oracle Linux 8 ARM
SSH_PUBLIC_KEY_PATH=/home/opc/.ssh/authorized_keys
```

### Full E2E Workflow Summary

1. **Auth**: Ensure API key auth works (`oci os ns get`)
2. **Find VM**: List instances, identify target
3. **Restart if needed**: OCI Mobile app → Compute → Instance → Restart → Wait for RUNNING + 30s
4. **SSH**: `ssh oci` (with retries)
5. **If SSH fails**: Use serial console to fix `sshd` + SELinux
6. **Deploy script**: Install PHP/Composer → Clone repo → Configure `.env` → Test → Cron
7. **Monitor**: Check logs, verify script runs every 5 min

This workflow handles the full lifecycle from instance management to automated capacity polling for Ampere A1 Flex in us-sanjose-1.