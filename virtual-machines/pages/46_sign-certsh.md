**Location:** `/home/ca/ca/sign-cert.sh` on the CA VM (`ca.internal`, IP `192.168.x.x`)

---

## Purpose
Non-interactive script to sign Certificate Signing Requests (CSRs) using the internal OpenSSL CA. Designed for automation pipelines where no manual passphrase entry is required (when the CA key is not encrypted).

---

## Functions

- **Parameter Validation:** Checks that CSR input file and certificate output path are provided.
- **CSR Signing:** Uses `openssl ca` with specified configuration to sign the CSR.
- **Batch Mode:** Runs OpenSSL CA in batch mode to suppress interactive prompts (except for passphrase if encrypted).

---

## Process Flow

1. Validate input parameters (`<csr-file>` and `<cert-outfile>` and `<hostname>` and `<ip address>`).
2. Call OpenSSL CA with `-batch` and appropriate config to sign the CSR.
3. Output the signed certificate to the specified location.
4. Exit with usage message if parameters are missing.

---




```bash
#!/usr/bin/env bash
# sign-cert.sh - Non-interactive OpenSSL CA signing with dynamic SAN injection + archiving
# Usage: ./sign-cert.sh <csr-file> <cert-outfile> <hostname> [ip-address]
set -euo pipefail

CA_DIR="/home/ca/ca"
OPENSSL_CNF="$CA_DIR/openssl.cnf"

CSR_FILE="${1:?csr required}"
CERT_OUT="${2:?out path required}"
HOSTNAME="${3:?hostname/CN required}"
IPADDR="${4:-}"

TS="$(date -u +%Y%m%d-%H%M%S)"
umask 077

# Archive dirs
ARC_ROOT="/home/ca/archive"
ARC_KEYS="$ARC_ROOT/keys"
ARC_CSRS="$ARC_ROOT/csrs"
ARC_CERTS="$ARC_ROOT/certs"
ARC_LATEST="$ARC_ROOT/latest"
mkdir -p "$ARC_KEYS" "$ARC_CSRS" "$ARC_CERTS" "$ARC_LATEST"

# Locate CA db + new_certs_dir
NEW_DIR=$(awk -F= '/^new_certs_dir/{print $2}' "$OPENSSL_CNF" | xargs)
DB=$(awk -F= '/^database/{print $2}' "$OPENSSL_CNF" | xargs)

# If CSR has a matching key next to it, archive the key as well
KEY_GUESS="$(dirname "$CSR_FILE")/${HOSTNAME}-${TS}.key"
[[ -f "$KEY_GUESS" ]] || KEY_GUESS=""

# Build SAN extfile
EXTFILE="$(mktemp)"
cat > "$EXTFILE" <<EOF
[ server_cert ]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = $HOSTNAME
EOF
[[ -n "$IPADDR" ]] && echo "IP.1 = $IPADDR" >> "$EXTFILE"

# Serialize CA ops
LOCKFILE="$CA_DIR/.ca_lock"
exec 9>"$LOCKFILE"
flock 9

# Sign CSR (OpenSSL will write canonical copy to new_certs_dir/<serial>.pem)
openssl ca -config "$OPENSSL_CNF" -extensions server_cert -extfile "$EXTFILE" \
  -in "$CSR_FILE" -out "$CERT_OUT" -batch

# Discover serial for this CN (last matching)
SERIAL="$(awk -v cn="CN=${HOSTNAME}$" '$4 ~ cn {s=$2} END{print s}' "$DB")"
CERT_CANON="$NEW_DIR/${SERIAL}.pem"

# Archive CSR
ARC_CSR="$ARC_CSRS/${HOSTNAME}-${TS}.csr"
cp -n "$CSR_FILE" "$ARC_CSR"

# Archive key if we can find it (optional)
if [[ -n "$KEY_GUESS" && -f "$KEY_GUESS" ]]; then
  ARC_KEY="$ARC_KEYS/${HOSTNAME}-${TS}.key"
  cp -n "$KEY_GUESS" "$ARC_KEY"
fi

# Archive cert (prefer canonical serial-named file)
ARC_CERT="$ARC_CERTS/${HOSTNAME}-${TS}-${SERIAL}.crt"
if [[ -f "$CERT_CANON" ]]; then
  cp -n "$CERT_CANON" "$ARC_CERT"
else
  cp -n "$CERT_OUT" "$ARC_CERT"
fi
chmod 0644 "$ARC_CERT"

# Update latest symlinks
ln -sfn "$ARC_CERT" "$ARC_LATEST/${HOSTNAME}.crt"
ln -sfn "$ARC_CSR"  "$ARC_LATEST/${HOSTNAME}.csr"
[[ -n "${ARC_KEY:-}" ]] && ln -sfn "$ARC_KEY" "$ARC_LATEST/${HOSTNAME}.key"

rm -f "$EXTFILE"

echo "Issued CN=$HOSTNAME serial=$SERIAL"
echo "Archived:"
echo "  CSR:  $ARC_CSR"
[[ -n "${ARC_KEY:-}" ]] && echo "  Key:  $ARC_KEY"
echo "  Cert: $ARC_CERT"
```
