# OCI Network & Security List Operations

## Discover VCN and Security Lists

### Get Root Compartment
```bash
ROOT_COMP=$(oci iam compartment list --all --compartment-id-in-subtree true \
  --query "data[?id=='compartment-id'].id | [0]" --raw-output --auth security_token)
```

### List All VCNs
```bash
oci network vcn list --compartment-id $ROOT_COMP --auth security_token
```

### List Security Lists for a VCN
```bash
oci network security-list list --compartment-id $ROOT_COMP --vcn-id <vcn_ocid> --auth security_token
```

### Get Specific Security List Details
```bash
oci network security-list get --security-list-id <security_list_ocid> --auth security_token
```

## Add SSH (Port 22) Ingress Rule

### Update Security List to Allow SSH
```bash
oci network security-list update --security-list-id <security_list_ocid> \
  --ingress-security-rules '[{"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"max":22,"min":22}},"description":"Allow SSH from anywhere"}]' \
  --force --auth security_token
```

### Alternative: Add Rule Without Overwriting Existing
```bash
# Get current rules first
CURRENT_RULES=$(oci network security-list get --security-list-id <security_list_ocid> --auth security_token --query "data.\"ingress-security-rules\"")

# Add new rule to existing
oci network security-list update --security-list-id <security_list_ocid> \
  --ingress-security-rules "$(echo $CURRENT_RULES | jq '. + [{"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"max":22,"min":22}},"description":"Allow SSH from anywhere"}]')" \
  --force --auth security_token
```

## Network Security Group (NSG) Alternative

### List NSGs
```bash
oci network nsg list --compartment-id $ROOT_COMP --vcn-id <vcn_ocid> --auth security_token
```

### Add SSH Rule to NSG
```bash
oci network nsg update --nsg-id <nsg_ocid> \
  --ingress-security-rules '[{"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"max":22,"min":22}},"description":"Allow SSH"}]' \
  --force --auth security_token
```

## Verify SSH Port is Open

### Test from OCI Cloud Shell or External
```bash
# From cloud shell
nc -zv <public_ip> 22

# Or check with nmap
nmap -p 22 <public_ip>
```

## Complete Fix Workflow for SSH Timeout

```bash
#!/bin/bash
# fix-ssh.sh - Fix SSH access to VM by adding port 22 to security list

ROOT_COMP=$(oci iam compartment list --all --compartment-id-in-subtree true \
  --query "data[?id=='compartment-id'].id | [0]" --raw-output --auth security_token)

echo "Root compartment: $ROOT_COMP"

# 1. Find VCN
VCN=$(oci network vcn list --compartment-id $ROOT_COMP --auth security_token \
  --query "data[0].id" --raw-output)
echo "VCN: $VCN"

# 2. Find Security Lists
SEC_LISTS=$(oci network security-list list --compartment-id $ROOT_COMP --vcn-id $VCN --auth security_token \
  --query "data[*].id" --raw-output)
echo "Security lists: $SEC_LISTS"

# 3. Update each security list
for SL in $SEC_LISTS; do
  echo "Updating security list: $SL"
  oci network security-list update --security-list-id $SL \
    --ingress-security-rules '[{"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"max":22,"min":22}},"description":"Allow SSH from anywhere"}]' \
    --force --auth security_token
done

# 4. Also check NSGs
NSGS=$(oci network nsg list --compartment-id $ROOT_COMP --vcn-id $VCN --auth security_token \
  --query "data[*].id" --raw-output)
for NSG in $NSGS; do
  echo "Updating NSG: $NSG"
  oci network nsg update --nsg-id $NSG \
    --ingress-security-rules '[{"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"max":22,"min":22}},"description":"Allow SSH"}]' \
    --force --auth security_token
done

echo "Done. Test: ssh oci"
```