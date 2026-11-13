# OCI Authentication

## CLI-First Principle

**OCI CLI is the primary tool for all operations. Phone/Buddy automation is FALLBACK ONLY.**

- Use `oci` CLI commands for: instance management, SSH, authentication, capacity checks
- Only use Buddy (phone automation) when CLI is unavailable or fails
- This ensures speed, scriptability, and reliability

---

## CRITICAL: Stop Spamming Auth Links

**Once auth is confirmed successful (token exists + verification passes), STOP. Do NOT run `oci session authenticate` again.**

```bash
# CHECK FIRST — if token works, DONE
oci os ns get --auth security_token && echo "AUTH OK" && exit 0

# Only if check fails: wipe + fresh auth ONCE
rm -rf /root/.oci/sessions/DEFAULT/token /root/.oci/sessions/DEFAULT/oci_session_token_* 2>/dev/null
oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera
oci os ns get --auth security_token
```

---

## Fresh Auth — One Command (Run Directly)

```bash
# 1. Wipe expired tokens
rm -rf /root/.oci/sessions/DEFAULT/token /root/.oci/sessions/DEFAULT/oci_session_token_* 2>/dev/null

# 2. Run auth — browser auto-launches on Termux host, prints URL in proot-distro
oci session authenticate --region us-sanjose-1 --tenancy-name diazolvera

# 3. Verify
oci os ns get --auth security_token
```

That's it. Run this command. If browser opens → log in → done. If URL prints → open it manually → done.

---

## xdg-open Fix (Run Once on Termux Host)

```bash
pkg install -y termux-api
ln -sf $(which termux-open) $(which xdg-open)
```

This enables browser auto-launch on real Termux. In proot-distro, URL prints instead.

---

## Persistent API Key (Optional — Works Everywhere, No Browser)

```bash
openssl genrsa -out /root/.oci/oci_api_key.pem 2048
openssl rsa -pubout -in /root/.oci/oci_api_key.pem -out /root/.oci/oci_api_key_public.pem
oci iam api-key upload --key-file /root/.oci/oci_api_key_public.pem
USER_OCID=$(oci iam user list --all --query "data[0].id" --raw-output)
FINGERPRINT=$(openssl rsa -pubout -outform DER -in /root/.oci/oci_api_key.pem | openssl md5 -c | cut -d' ' -f2)
TENANCY_OCID=$(oci iam tenancy get --query "data.id" --raw-output)
printf "[DEFAULT]\nuser=%s\nfingerprint=%s\ntenancy=%s\nregion=us-sanjose-1\nkey_file=/root/.oci/oci_api_key.pem\n" "$USER_OCID" "$FINGERPRINT" "$TENANCY_OCID" > /root/.oci/config
chmod 600 /root/.oci/config
oci os ns get
```

---

## Quick Check

```bash
oci os ns get --auth security_token  # session token
oci os ns get                        # API key
```