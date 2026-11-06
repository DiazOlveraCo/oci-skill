# Stage 02: Core Operations

## Inputs
| Artifact | Location | Format |
|----------|----------|--------|
| Valid auth | Stage 01 output | Session token + config |
| Compartment OCID | User input / discovery | OCID string |
| Region | `~/.oci/config` | String |

## Process
Execute OCI CLI commands for resource management.

## Outputs
| Artifact | Location | Format |
|----------|----------|--------|
| Instance list | `output/instances.json` | JSON |
| Compartment list | `output/compartments.json` | JSON |
| AD list | `output/availability-domains.json` | JSON |

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

### Launch ARM Free Tier Instance
```bash
oci compute instance launch \
  --compartment-id <compartment_ocid> \
  --availability-domain <ad_name> \
  --subnet-id <subnet_ocid> \
  --image-id <image_ocid> \
  --shape "VM.Standard.A1.Flex" \
  --shape-config '{"ocpus":4,"memoryInGBs":24}' \
  --display-name "arm-free-$(date +%s)" \
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

## ARM Capacity Loop (Free Tier)
```bash
while true; do
  oci compute instance launch \
    --compartment-id $COMP --availability-domain $AD \
    --subnet-id $SUBNET --image-id $IMAGE \
    --shape "VM.Standard.A1.Flex" \
    --shape-config '{"ocpus":4,"memoryInGBs":24}' \
    --display-name "arm-free-$(date +%s)" \
    --metadata '{"ssh_authorized_keys":"'"$SSH_KEY"'"}' \
    --wait-for-state RUNNING --max-wait-seconds 600 \
    --auth security_token && break
  echo "Capacity full, retrying in 60s..."
  sleep 60
done
```

## Validation
```bash
# Check instance state
oci compute instance list --compartment-id $COMP --auth security_token \
  --query "data[*].{name:\"display-name\", state:\"lifecycle-state\", id:id, shape:\"shape\", ad:\"availability-domain\"}"
```