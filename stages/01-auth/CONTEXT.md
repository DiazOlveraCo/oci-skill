# Stage 01: Authentication

## Inputs
| Artifact | Location | Format |
|----------|----------|--------|
| Tenancy name | User input | String |
| Region | User input | String (e.g., us-sanjose-1) |
| Existing config | `~/.oci/config` | INI |

## Process
1. Run browser OAuth flow
2. Create session token + keys in `~/.oci/sessions/DEFAULT/`
3. Update `~/.oci/config` with profile

## Outputs
| Artifact | Location | Format |
|----------|----------|--------|
| Session token | `~/.oci/sessions/DEFAULT/token` | JWT |
| Private key | `~/.oci/sessions/DEFAULT/oci_api_key.pem` | PEM |
| Public key | `~/.oci/sessions/DEFAULT/oci_api_key_public.pem` | PEM |
| Updated config | `~/.oci/config` | INI |

## Commands
```bash
# Interactive browser auth (works on Termux/Android)
oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera
# Enter profile name when prompted (e.g., DEFAULT)

# Refresh expired token
oci session authenticate --profile-name DEFAULT --region us-sanjose-1 --auth security_token

# API key setup (headless)
oci setup config
```

## Validation
```bash
oci os ns get --auth security_token
# Returns namespace = success
```

## Termux/Android Fix
```bash
ln -sf $PREFIX/bin/termux-open-url $PREFIX/bin/xdg-open
```