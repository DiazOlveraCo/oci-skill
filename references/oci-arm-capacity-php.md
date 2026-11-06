# OCI Ampere VM Capacity Automation (PHP)

## Project: hitrov/oci-arm-host-capacity

**Repository**: https://github.com/hitrov/oci-arm-host-capacity
**Language**: PHP 7.x / 8.x + Composer
**License**: MIT
**Stars**: 2

## Assessment: Not Overkill — Lightweight & Simple

This is a **minimal, single-purpose script** — not overkill. It's actually the simplest option:
- One PHP file + composer install
- No heavy SDK or framework
- Direct REST API calls to OCI LaunchInstance endpoint
- Runs via cron or GitHub Actions
- `.env` file configuration

## Prerequisites

```bash
# PHP 7.x or 8.x
php -v

# Composer
composer -V
```

## Installation

```bash
git clone https://github.com/hitrov/oci-arm-host-capacity.git
cd oci-arm-host-capacity/
composer install
cp .env.example .env
```

## Configuration (.env)

```ini
# Region
OCI_REGION=us-sanjose-1

# Authentication (from API key generation)
OCI_TENANCY_ID=ocid1.tenancy.oc1..aaaaaaaasimpzm2iolrderse5ostt5l5djgsdt3slzmmeqzfbluv3ruqmugq
OCI_USER_ID=ocid1.user.oc1..aaaaaaaake25vifglqspfashsmyionbxqt2tc65tbpd2a3qiixiv7exaskxa
OCI_FINGERPRINT=d8:9d:a1:15:8a:05:61:ad:5b:4b:2a:44:a4:be:03:9d
OCI_PRIVATE_KEY_PATH=/root/.oci/oci_api_key.pem

# Instance parameters
OCI_AVAILABILITY_DOMAIN=  # leave blank to auto-detect
OCI_SUBNET_ID=ocid1.subnet.oc1.us-sanjose-1.xxxxx
OCI_IMAGE_ID=ocid1.image.oc1.us-sanjose-1.xxxxx
OCI_SSH_PUBLIC_KEY="ssh-rsa AAAA..."

# Shape (Ampere A1 Flex)
OCI_SHAPE=VM.Standard.A1.Flex
OCI_OCPUS=4
OCI_MEMORY_IN_GBS=24

# Optional: max instances check
OCI_MAX_INSTANCES=1
```

## How It Works

1. Calls `ListAvailabilityDomains` OCI API
2. Calls `ListInstances` to check existing instances (avoids exceeding limits)
3. Calls `LaunchInstance` with your parameters
4. On "Out of capacity" → retries on next run
5. On success → exits

## Running

### Manual
```bash
php oci-arm-host-capacity.php
```

### Cron (every 5 minutes)
```bash
*/5 * * * * cd /path/to/oci-arm-host-capacity && php oci-arm-host-capacity.php >> /var/log/oci-arm.log 2>&1
```

### GitHub Actions (cloud, no local PHP needed)
See `.github/workflows/` in the repo — runs on GitHub's servers.

## Pros vs Python Alternatives

| Feature | PHP (hitrov) | Python (I-l-y-a-Z-z) | Python (praveenkarunarathne) |
|---------|-------------|---------------------|------------------------------|
| Dependencies | PHP + Composer | Python + oci SDK | Python + oci SDK + Docker |
| Stars | 2 | 4 | 28 |
| Notifications | None | None | Telegram |
| Docker | No | No | Yes |
| Retry logic | Cron-based | Built-in 60s loop | Built-in with backoff |
| Official SDK | Custom PHP SDK | Official OCI SDK | Official OCI SDK |
| Complexity | Low | Low | Medium |

## When to Use PHP Version
- You prefer PHP over Python
- You want cron-based scheduling (cleaner than background loops)
- You want minimal dependencies
- You don't need Telegram/Docker features

## When to Use Python Instead
- You want built-in retry loops (no cron setup)
- You want Telegram notifications
- You want Docker deployment
- You prefer official OCI SDK

## Troubleshooting

### "Out of host capacity" (expected)
Normal. Script will retry on next cron run. Capacity appears intermittently.

### "NotAuthenticated"
API key not registered in OCI Console. Add public key: Profile → User Settings → API Keys.

### "LimitExceeded"
You've hit free tier limits. Check: OCI Console → Governance → Limits.

### Service limits for A1
- Free tier: 4 OCPU + 24 GB RAM total across all A1 instances
- If you already have an instance, reduce OCPUS/MEMORY to fit remaining quota

## Notes for San Jose (us-sanjose-1)
- San Jose is a **popular region** with high demand for free tier
- Consider trying multiple Availability Domains
- PAYG (Pay As You Go) upgrade gives priority access to capacity
- Set budget alerts to avoid unexpected charges on PAYG
