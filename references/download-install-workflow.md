# Download and Install Workflow (Generic — Linux VM)

## Scope

This page describes **only** the general workflow for downloading files to a Linux
VM with `curl` and installing them locally. It applies to any artifact type
(RPM, DEB, tarball, standalone binary, script bundle).

It contains **no package names, versions, or vendor-specific URLs**. Resolve
those at runtime from the current repository or release page — never hardcode
them into skill documentation.

---

## 1. Prepare a Download Directory

```bash
mkdir -p /tmp/downloads
cd /tmp/downloads
```

Keep downloads isolated so cleanup is unambiguous.

---

## 2. Download with curl

### Basic download

```bash
curl -fL -o /tmp/downloads/<artifact> "<url>"
```

- `-f` — fail on HTTP errors instead of saving an error page
- `-L` — follow redirects (release URLs commonly redirect)
- `-o` — write to an explicit filename

### Large files (resumable, retried, visible progress)

```bash
curl -fL -C - \
  --retry 5 --retry-delay 5 --retry-all-errors \
  --connect-timeout 30 \
  --progress-bar \
  -o /tmp/downloads/<artifact> "<url>"
```

- `-C -` — resume a partial download instead of starting over
- `--retry*` — survive transient network failures
- `--progress-bar` — visible progress without full verbose noise
- Use `-v` only when diagnosing a failure, not for routine downloads

### Long downloads must not block the session

Run large downloads in the background, then poll the file size with `-C -`
to resume if interrupted. Do not let a download outlive a command timeout with
no resume path.

---

## 3. Verify the Download

```bash
ls -lh /tmp/downloads/<artifact>
file /tmp/downloads/<artifact>

# When the publisher provides a checksum:
sha256sum /tmp/downloads/<artifact>
# compare against the published value
```

A truncated file, an HTML error page, or a mismatched checksum must be treated
as a failed download — delete and re-download.

---

## 4. Install by Artifact Type

### RPM (Oracle Linux / RHEL / Fedora)

```bash
# Preferred: resolves dependencies against enabled repos and local files
dnf localinstall -y /tmp/downloads/<package>.rpm

# Fallback: no dependency resolution
rpm -ivh /tmp/downloads/<package>.rpm
```

If dependencies are missing, download them first (one at a time with `curl`,
or via a dependency-aware tool if the environment allows), then install the
whole set together.

### DEB (Debian / Ubuntu)

```bash
apt install -y /tmp/downloads/<package>.deb
```

### Tarball

```bash
# Inspect before extracting
tar -tf /tmp/downloads/<archive>.tar.gz | head

# Extract into a staging directory, then move into place
tar -xzf /tmp/downloads/<archive>.tar.gz -C /opt
```

### Standalone binary

```bash
install -m 755 /tmp/downloads/<binary> /usr/local/bin/<binary>
```

### Script bundle

Inspect first, then run only if the source is trusted:

```bash
less /tmp/downloads/<script>.sh
bash /tmp/downloads/<script>.sh
```

---

## 5. Verify the Installation

```bash
command -v <binary>
<binary> --version
```

Do not report success until the installed command runs.

---

## 6. Clean Up

```bash
rm -rf /tmp/downloads
```

---

## Failure Handling

| Symptom | Cause | Action |
|---------|-------|--------|
| `curl: (22)` | HTTP 4xx/5xx | Check the URL and that the artifact still exists |
| File size smaller than expected | Interrupted transfer | Re-run with `-C -` to resume |
| `file` reports HTML | Saved an error/redirect page | Re-download with `-fL` |
| Checksum mismatch | Corrupt or wrong file | Delete and re-download from the canonical source |
| `Failed dependencies` | Missing RPM deps | Download dependencies, install them together |
| `Permission denied` | Not executable / wrong owner | `install -m 755` or run with the required privileges |

---

## Rules

1. Download with `curl`; install locally from the downloaded artifact.
2. Never hardcode package names, version numbers, or vendor URLs in a skill —
   resolve the current artifact at runtime.
3. Verify before installing; verify again after.
4. Treat this page as the single source for the download/install mechanics;
   product-specific pages only supply the artifact name and URL.
