# OCI CLI Installation via RPM (curl + local install)

## Core Rule

**Prefer downloading RPMs with `curl` and installing locally with `rpm` or `dnf localinstall` instead of using `dnf install` directly.** This provides:

- Full visibility into download progress and speed
- Offline installation capability
- Explicit control over package versions and sources
- No dependency on enabled repositories (works on minimal installs)

---

## Procedure

### 1. Download Main Packages

```bash
# Create download directory
mkdir -p /tmp/rpm-downloads

# Download OCI CLI and SDK RPMs (latest from Oracle Linux 9 developer repo)
curl -L -o /tmp/rpm-downloads/python39-oci-cli-3.95.1-1.0.1.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python39-oci-cli-3.95.1-1.0.1.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/python39-oci-sdk-2.189.0-1.0.1.el9.aarch64.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python39-oci-sdk-2.189.0-1.0.1.el9.aarch64.rpm"
```

### 2. Download Python Dependencies

Get all required Python dependencies from Oracle Linux repos:

```bash
# From developer repo (aarch64)
curl -L -o /tmp/rpm-downloads/deps/python3-arrow-1.1.0-2.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python3-arrow-1.1.0-2.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-circuitbreaker-1.3.2-1.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python3-circuitbreaker-1.3.2-1.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-certifi-2020.12.5-4.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python3-certifi-2020.12.5-4.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-jmespath-0.10.0-4.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python3-jmespath-0.10.0-4.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-prompt-toolkit-3.0.38-4.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python3-prompt-toolkit-3.0.38-4.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-terminaltables-3.1.10-8.0.1.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python3-terminaltables-3.1.10-8.0.1.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-importlib-metadata-4.12.0-2.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python3-importlib-metadata-4.12.0-2.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-typing-extensions-4.14.1-1.0.2.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python3-typing-extensions-4.14.1-1.0.2.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-wcwidth-0.2.5-7.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python3-wcwidth-0.2.5-7.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-zipp-3.20.1-2.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/getPackage/python3-zipp-3.20.1-2.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-pyOpenSSL-21.0.0-1.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/developer/EPEL/aarch64/getPackage/python3-pyOpenSSL-21.0.0-1.el9.noarch.rpm"

# From appstream repo (aarch64)
curl -L -o /tmp/rpm-downloads/deps/python3-cryptography-36.0.1-5.el9_6.aarch64.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/appstream/aarch64/getPackage/python3-cryptography-36.0.1-5.el9_6.aarch64.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-pytz-2021.1-5.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/appstream/aarch64/getPackage/python3-pytz-2021.1-5.el9.noarch.rpm"

# From baseos repo (aarch64)
curl -L -o /tmp/rpm-downloads/deps/python3-pyyaml-5.4.1-6.0.1.el9.aarch64.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/aarch64/getPackage/python3-pyyaml-5.4.1-6.0.1.el9.aarch64.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-urllib3-1.26.5-8.el9_8.2.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/aarch64/getPackage/python3-urllib3-1.26.5-8.el9_8.2.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-cffi-1.14.5-5.0.1.el9.aarch64.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/aarch64/getPackage/python3-cffi-1.14.5-5.0.1.el9.aarch64.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-pycparser-2.20-6.0.1.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/aarch64/getPackage/python3-pycparser-2.20-6.0.1.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-ply-3.11-14.0.1.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/aarch64/getPackage/python3-ply-3.11-14.0.1.el9.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-idna-2.10-8.el9_8.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/aarch64/getPackage/python3-idna-2.10-8.el9_8.noarch.rpm"

curl -L -o /tmp/rpm-downloads/deps/python3-pysocks-1.7.1-12.0.1.el9.noarch.rpm \
  "https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/aarch64/getPackage/python3-pysocks-1.7.1-12.0.1.el9.noarch.rpm"
```

### 3. Install All Packages

```bash
# Install all dependencies first
rpm -ivh /tmp/rpm-downloads/deps/*.rpm

# Install main OCI packages
rpm -ivh /tmp/rpm-downloads/python39-oci-cli-3.95.1-1.0.1.el9.noarch.rpm \
         /tmp/rpm-downloads/python39-oci-sdk-2.189.0-1.0.1.el9.aarch64.rpm

# If any dependency conflicts, use --nodeps after dependencies are satisfied
# rpm -ivh --nodeps /tmp/rpm-downloads/python39-oci-sdk-2.189.0-1.0.1.el9.aarch64.rpm
```

### 4. Verify Installation

```bash
oci --version
# Expected: 3.95.1 (or latest)
```

---

## PowerShellAI Integration

For automated installation with robust error handling, use the PowerShell script:

```powershell
# From the OCI skill scripts directory
& "/root/.config/opencode/skills/oci/scripts/Ensure-OCICLI.ps1" -Verbose -PackageManagerTimeout 300 -ScriptTimeout 600
```

The script:
- Uses `curl` for all downloads (shows progress/speed)
- Installs via `rpm` or `dnf localinstall`
- Handles retries with AI-assisted error analysis
- Creates GitHub issue on failure
- Verifies with `oci --version`

---

## When to Use This Approach

- **Minimal/offline systems** — no dnf repos enabled
- **Specific version pinning** — exact RPM URLs ensure version consistency
- **Air-gapped environments** — download on connected machine, transfer, install offline
- **CI/CD pipelines** — deterministic downloads with explicit URLs
- **Troubleshooting** — inspect RPM contents before install with `rpm -qlp package.rpm`

---

## Notes

- Package versions change — check `https://yum.oracle.com/repo/OracleLinux/OL9/developer/aarch64/index.html` for latest
- Architecture: use `aarch64` for Ampere/ARM, `x86_64` for Intel/AMD
- The `python39-oci-sdk` package is ~83 MB — expect 2-5 min download on typical connections
- All Python dependencies come from Oracle Linux baseos/appstream/developer/EPEL repos