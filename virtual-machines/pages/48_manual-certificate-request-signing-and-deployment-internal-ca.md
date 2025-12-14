## Purpose  
Generate a Certificate Signing Request (CSR) with full Distinguished Name (DN) fields for submission to an internal Certificate Authority (CA). Ensures certificates issued include essential identification fields (CN, O, OU) and are trusted across your environment.

---

## Usage Context  
Run this on the system where the certificate/key will be used (e.g., server, appliance, VM).

---

## Command Syntax  

```bash
openssl req -new -newkey rsa:4096 -nodes \
  -keyout <path-to-private-key> \
  -out <path-to-csr> \
  -subj "/CN=<common-name>/O=<organization>/OU=<organizational-unit>"
````

* Replace `<path-to-private-key>` with your desired private key output location.
* Replace `<path-to-csr>` with your desired CSR output location.
* Replace `<common-name>` with the FQDN or hostname (e.g., `server.example.com`).
* Replace `<organization>` with your organization name (e.g., `MyCompany`).
* Replace `<organizational-unit>` with the department or unit name (e.g., `IT`).

---

## Next Steps: Sign and Deploy the Certificate Manually

---

### 1. **Transfer CSR to CA**

From the server that generated the CSR:

```bash
scp <path-to-csr> ca@ca.cfolino.com:/home/ca/ca/pending-csrs/
```

Example:

```bash
scp /etc/ssl/certs/server.csr ca@ca.cfolino.com:/home/ca/ca/pending-csrs/
```

---

### 2. **Sign the Certificate on the CA**

SSH into the CA VM:

```bash
ssh ca@ca.cfolino.com
```

Run the signing script:

```bash
/home/ca/ca/sign-cert.sh /home/ca/ca/pending-csrs/<csr-file> /home/ca/ca/signed-certs/<cert-file> <hostname> [ip-address]
```

Example:

```bash
/home/ca/ca/sign-cert.sh /home/ca/ca/pending-csrs/server.csr /home/ca/ca/signed-certs/server.crt server.cfolino.com 192.168.10.128
```

---

### 3. **Copy the Signed Certificate to the Server**

From the CA:

```bash
scp /home/ca/ca/signed-certs/<cert-file> user@target-server:/etc/ssl/certs/
scp /home/ca/ca/private/<key-file> user@target-server:/etc/ssl/private/
```

Example:

```bash
scp /home/ca/ca/signed-certs/server.crt root@proxmox:/etc/ssl/certs/
scp /home/ca/ca/private/server.key root@proxmox:/etc/ssl/private/
```

---

### 4. **Deploy the Certificate**

#### Proxmox VE (PVE):

```bash
cp /etc/ssl/certs/server.crt /etc/pve/nodes/proxmox/pve-ssl.pem
cp /etc/ssl/private/server.key /etc/pve/nodes/proxmox/pve-ssl.key
systemctl restart pveproxy
```

#### Nginx:

```bash
cp server.crt /etc/nginx/ssl/server.crt
cp server.key /etc/nginx/ssl/server.key
systemctl reload nginx
```

#### Apache:

```bash
cp server.crt /etc/apache2/ssl/server.crt
cp server.key /etc/apache2/ssl/server.key
systemctl reload apache2
```

---

### 5. **Verify the Certificate**

From a client:

```bash
openssl s_client -connect <hostname>:443 -servername <hostname> -showcerts
```

Verify:

* Correct CN and SANs
* Trusted issuer (your internal CA)
* No browser or curl warnings

---

### 6. **Ensure CA Trust**

On client systems:

```bash
sudo cp ca.crt /usr/local/share/ca-certificates/cfolino-ca.crt
sudo update-ca-certificates
```

---

## Monitoring & Logs

Check for signing activity and errors in the CA log:

```bash
tail -f /home/ca/ca/logs/ca.log
```

Inspect issued certificates in:

```bash
ls /home/ca/ca/newcerts/
```

---

## Troubleshooting

* Ensure `openssl.cnf` contains the `[ server_cert ]` extension block.
* Confirm the CA key and certificate are valid and unlocked if encrypted.
* Check file permissions for the CA private key and CSR.
* If the script fails silently, run it with `bash -x` for debugging:

```bash
bash -x /home/ca/ca/sign-cert.sh /home/ca/ca/pending-csrs/server.csr /home/ca/ca/signed-certs/server.crt server.cfolino.com ip address
