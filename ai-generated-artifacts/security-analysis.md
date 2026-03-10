# Security Analysis: `copilot-install.sh`

---

## 🔴 HIGH Severity

### 1. Pipe-to-shell execution (curl/wget | bash)
**Line:** Usage comment (lines 5–6), and the script is **designed** to be run this way.
```sh
curl -fsSL https://gh.io/copilot-install | bash
```
This is the most dangerous pattern in shell scripting:
- The script executes **without the user being able to review it first**.
- A compromised CDN, DNS hijack, or MITM attack (despite HTTPS) allows arbitrary code execution on the user's machine.
- There is **no signature verification** of the script itself before execution.

**Recommendation:** Instruct users to download the script, inspect it, and then run it. Alternatively, provide a GPG-signed script with verification instructions.

---

### 2. Checksums downloaded from the same server as the binary
**Lines:** 41–42, 51–52, 61–62
```sh
CHECKSUMS_URL="https://github.com/github/copilot-cli/releases/latest/download/SHA256SUMS.txt"
```
The checksum file is fetched from the **same host** as the binary. If that host is compromised, both the binary and its checksum can be replaced simultaneously, making checksum validation ineffective.

**Recommendation:** Host the checksum file (and ideally a GPG signature of it) on a separate, independently controlled server or sign checksums with a GPG key whose public key is embedded in the install script.

---

### 3. Checksum validation is non-fatal (silently skipped)
**Lines:** 92–95
```sh
else
  echo "Warning: No sha256sum or shasum found, skipping checksum validation."
fi
```
If neither `sha256sum` nor `shasum` is available, the script continues **without any integrity check** and installs the (potentially tampered) binary. A warning to stderr is insufficient for a security control.

**Recommendation:** Treat missing checksum tools as a fatal error, or verify integrity via another means (e.g., embedded hash comparison using `openssl dgst`).

---

## 🟠 MEDIUM Severity

### 4. Unvalidated `VERSION` environment variable used in URLs
**Lines:** 54–63
```sh
VERSION="$(git ls-remote --tags https://github.com/github/copilot-cli | tail -1 | awk -F/ '{print $NF}')"
...
DOWNLOAD_URL="https://github.com/github/copilot-cli/releases/download/${VERSION}/copilot-..."
```
The `VERSION` variable is taken directly from user-controlled environment input and from `git ls-remote` output, then embedded into a URL without sanitization. While the URL is passed to `curl`/`wget` (not `eval`), a maliciously crafted version string (e.g. with `@`, `#`, or other URL-special characters) could redirect the download to an attacker-controlled server or alter the request.

**Recommendation:** Validate `VERSION` against a strict pattern (e.g., `^v?[0-9]+\.[0-9]+\.[0-9]+.*$`) before use.

---

### 5. `tar` extraction without `--strip-components` or explicit member list
**Line:** 131
```sh
tar -xz -C "$INSTALL_DIR" -f "$TMP_TARBALL"
```
Extracting a tarball without restricting which paths are written is a classic **tar path traversal** vulnerability. A malicious or tampered archive could contain entries with absolute paths (e.g., `/etc/cron.d/evil`) or `../` components that escape `$INSTALL_DIR`.

**Recommendation:** Either extract to a temp directory and move only the expected binary, or use `--strip-components` and explicitly name the expected file:
```sh
tar -xz -C "$TMP_DIR" -f "$TMP_TARBALL"
mv "$TMP_DIR/copilot" "$INSTALL_DIR/copilot"
```

---

### 6. `PREFIX` environment variable is fully user-controlled without sanitization
**Lines:** 113–114
```sh
PREFIX="${PREFIX:-/usr/local}"
INSTALL_DIR="$PREFIX/bin"
```
A user (or a compromised parent process) could set `PREFIX` to a path that causes the script to install the binary to an unexpected location (e.g., `PREFIX=/tmp/evil` or a path with spaces that breaks subsequent commands if not quoted properly). While variables are quoted here, the directory is `mkdir -p`'d without any restrictions.

**Recommendation:** Validate that `PREFIX` is an absolute path and does not contain suspicious characters before use.

---

### 7. `git ls-remote` output used without verification
**Line:** 48
```sh
VERSION="$(git ls-remote --tags https://github.com/github/copilot-cli | tail -1 | awk -F/ '{print $NF}')"
```
`git ls-remote` is run over HTTPS, but the result is taken as-is with no format validation. The `awk` processing slightly limits what passes through, but a MITM or compromised Git server could still inject a crafted version string used to construct a URL.

**Recommendation:** Sanitize the result of `git ls-remote` with a strict regex before trusting it.

---

## 🟡 LOW Severity

### 8. `rm -rf "$TMP_DIR"` not called on all error paths
**Lines:** 101–105
```sh
if ! tar -tzf "$TMP_TARBALL" >/dev/null 2>&1; then
  echo "Error: ..."
  rm -rf "$TMP_DIR"
  exit 1
fi
```
The `TMP_DIR` cleanup is manually repeated in several error paths. If a new error path is added in the future and cleanup is missed, **downloaded binaries are left in `/tmp`**, potentially containing sensitive artifacts.

**Recommendation:** Use a `trap` to guarantee cleanup on exit:
```sh
trap 'rm -rf "$TMP_DIR"' EXIT
```

---

### 9. `$SHELL` variable used to determine shell RC file
**Lines:** 143–147
```sh
case "$(basename "${SHELL:-/bin/sh}")" in
  zsh)  RC_FILE="$HOME/.zshrc" ;;
  bash) RC_FILE="$HOME/.bashrc" ;;
  *)    RC_FILE="$HOME/.profile" ;;
esac
```
`$SHELL` can be overridden in the environment before running the script (e.g., via `SHELL=/bin/evil ./install.sh`). A crafted `$SHELL` value leading to a strange `$RC_FILE` path could cause the PATH export to be written somewhere unexpected.

**Recommendation:** Validate `$SHELL` against known values, or inform the user of the exact change and require explicit confirmation.

---

### 10. `set -e` is insufficient for all failure modes
**Line:** 2
```sh
set -e
```
`set -e` does not catch errors in all contexts (e.g., within `if` conditions, pipeline stages, or subshell command substitutions). For example:
```sh
VERSION="$(git ls-remote ... | tail -1 | awk ...)"
```
If `git ls-remote` fails mid-pipe, `set -e` may not abort the script.

**Recommendation:** Add `set -o pipefail` (already supported in bash) to catch failures in pipelines:
```sh
set -euo pipefail
```
This also catches uses of unset variables with `-u`.

---

## Summary Table

| # | Severity | Issue |
|---|----------|-------|
| 1 | 🔴 High | Script designed for pipe-to-shell execution |
| 2 | 🔴 High | Checksum and binary downloaded from same origin |
| 3 | 🔴 High | Missing checksum tool silently skips validation |
| 4 | 🟠 Medium | Unvalidated `VERSION` variable used in URLs |
| 5 | 🟠 Medium | `tar` extraction without path traversal protection |
| 6 | 🟠 Medium | `PREFIX` env var used without validation |
| 7 | 🟠 Medium | `git ls-remote` output used without sanitization |
| 8 | 🟡 Low | Temp directory not cleaned up via `trap` |
| 9 | 🟡 Low | `$SHELL` variable can be manipulated |
| 10 | 🟡 Low | `set -e` insufficient without `pipefail` |
