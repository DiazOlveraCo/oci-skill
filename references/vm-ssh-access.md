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
Host oci-vm
    HostName <public-ip>
    User opc
    IdentityFile ~/.ssh/id_rsa
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

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

## Current VM Targets
Run `oci compute instance list --all --query "data[?\"lifecycle-state\"=='RUNNING']"` to see running VMs.