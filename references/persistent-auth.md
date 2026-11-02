# Persistent Authentication (API Keys)

## Problem
Session tokens expire (~60 min). Need persistent auth for automation/SSH access.

## Solution: API Key Authentication

### Setup (One-time)
```bash
# 1. Generate keys (already done)
openssl genrsa -out ~/.oci/oci_api_key.pem 2048
chmod 600 ~/.oci/oci_api_key.pem
openssl rsa -pubout -in ~/.oci/oci_api_key.pem -out ~/.oci/oci_api_key_public.pem

# 2. Get fingerprint
openssl rsa -pubout -outform DER -in ~/.oci/oci_api_key.pem | openssl md5 -c
# Output: d8:9d:a1:15:8a:05:61:ad:5b:4b:2a:44:a4:be:03:9d

# 3. Register public key in OCI Console
# Console → Profile → User Settings → API Keys → "Add API Key" → Paste public key

# 4. Create persistent config
cat > ~/.oci/config << 'EOF'
[DEFAULT]
user=ocid1.user.oc1..<your-user-ocid>
fingerprint=<your-fingerprint>
tenancy=ocid1.tenancy.oc1..<your-tenancy-ocid>
region=us-sanjose-1
key_file=/root/.oci/oci_api_key.pem
EOF
chmod 600 ~/.oci/config
```

### Verify
```bash
oci os ns get
# Returns namespace = success (no --auth needed)
```

### Current Values
| Parameter | Value |
|-----------|-------|
| User OCID | `ocid1.user.oc1..aaaaaaaake25vifglqspfashsmyionbxqt2tc65tbpd2a3qiixiv7exaskxa` |
| Fingerprint | `d8:9d:a1:15:8a:05:61:ad:5b:4b:2a:44:a4:be:03:9d` |
| Tenancy OCID | `ocid1.tenancy.oc1..aaaaaaaasimpzm2iolrderse5ostt5l5djgsdt3slzmmeqzfbluv3ruqmugq` |
| Region | `us-sanjose-1` |
| Key file | `/root/.oci/oci_api_key.pem` |
| Public key | See `~/.oci/oci_api_key_public.pem` |

### Public Key (Register in Console)
```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAne2VfjwaiyMFsUWO4NH2
59859GUiMQ1sOAIvkib9omo7qE6KTDlrYyyPjRlG8sIG44YY9kC6T5o99ZLUhxUm
qnBTaYU1UawjwubDXmviKRp4r9pnq8f1A4hhJZ0lFMEwEM2xWNfbUepdcT85mpOO
0FI/muBm6sBL52qOZo5E5CsDT0pxLV2+0P2HtS9qHiL8gCaJlLJw623vYul6tqrH
V9gFQoqpl3UNomfMOhnYoMu+aXLodIuF5/Jsp5LGSxoJtFfRqRTcVC5nbexMnTe8
VBkeI2maTsneLIEXfBJ3KxRr5cOMyyvyZ29uc9dGLN+fl85L9Vq3F1NFDoGV/2Dj
EwIDAQAB
-----END PUBLIC KEY-----
```