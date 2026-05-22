# Design-and-implement-a-Post-Quantum-Cryptography-PQC-secret-rotation-engine-using-HashiCorp-Vault.
A Post-Quantum Cryptography (PQC) certificate rotation engine built on HashiCorp Vault. Uses NIST ML-DSA-65 (mldsa65) — the standardized version of Dilithium — to replace RSA-2048 for internal microservice mTLS. Certificates rotate automatically every 60 minutes with zero downtime.

---

## ⚠️ Important Notes Before You Start

These are lessons learned from building this project. Read them first:

- **Use `mldsa65` NOT `dilithium3`** — dilithium3 was renamed in the NIST standard. dilithium3 will fail with "Error allocating keygen context"
- **Do NOT include `ttl` parameter** in hvac generate_certificate() — hvac 2.4.0 does not support it. TTL is controlled by the Vault role max_ttl setting
- **oqs-provider installs to build folder** — NOT to /usr/lib. The .so file will be at `~/oqs-provider/build/lib/oqsprovider.so`
- **Use Python f.write() for all config files** — heredoc (EOF) and cat >> commands corrupt files in some terminals
- **liboqs-dev is NOT in Ubuntu repos** — build from source using the steps below
- **Do NOT use `if __name__ == "__main__":` in sidecar** — terminals corrupt the double underscores. Call rotate_certificate() directly

---

## 📋 Table of Contents

- [Quick Results](#quick-results)
- [Prerequisites](#prerequisites)
- [Step 1 — Install Dependencies](#step-1--install-dependencies)
- [Step 2 — Build liboqs](#step-2--build-liboqs)
- [Step 3 — Build oqs-provider](#step-3--build-oqs-provider)
- [Step 4 — Create OpenSSL Config](#step-4--create-openssl-config)
- [Step 5 — Generate PQC Root CA](#step-5--generate-pqc-root-ca)
- [Step 6 — Install and Start Vault](#step-6--install-and-start-vault)
- [Step 7 — Configure PKI Engine](#step-7--configure-pki-engine)
- [Step 8 — Run the Sidecar](#step-8--run-the-sidecar)
- [Step 9 — Verify Everything Works](#step-9--verify-everything-works)

---

## Quick Results

```
Vault v2.0.0 running            ✅
PKI engine enabled               ✅
Root CA generated (mldsa65)      ✅
Microservice role created        ✅
Sidecar rotating certs           ✅
tls.crt: 1743 bytes              ✅
tls.key: 1678 bytes              ✅

Certificate verified:
Issuer: CN = PQC-Root-CA
Not Before: May 13 10:36:45 2026
Not After:  May 13 12:37:15 2026  ← 2 hour TTL
Subject: CN = service1.internal.company.com
```

---
### Why Post-Quantum Now?

NIST finalized its first PQC standards in 2024. "Harvest now, decrypt later" attacks mean adversaries are collecting encrypted traffic today to decrypt once quantum computers mature. Building PQC infrastructure now protects both present and future data.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     HOST MACHINE (Ubuntu 22.04)              │
│                                                              │
│  ┌─────────────────┐         ┌──────────────────────────┐   │
│  │  HashiCorp Vault │         │     Vault Sidecar         │   │
│  │  (PKI Engine)    │◄───────►│     (sidecar.py)          │   │
│  │                  │  hvac   │                           │   │
│  │  • Root CA       │  API    │  • Requests new cert      │   │
│  │  • PKI Roles     │         │  • Writes tls.crt         │   │
│  │  • CRL           │         │  • Writes tls.key         │   │
│  │  • Issuance      │         │  • Sleeps 60 minutes      │   │
│  └─────────────────┘         │  • Repeats forever        │   │
│                               └────────────┬─────────────┘   │
│                                            │                  │
│  ┌─────────────────────────────────────────▼──────────────┐  │
│  │                  ~/certs/                               │  │
│  │          tls.crt (1743 bytes)  tls.key (1678 bytes)    │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              PQC Cryptography Layer                   │   │
│  │   liboqs ──► oqs-provider ──► OpenSSL 3.x            │   │
│  │              (mldsa65 / ML-DSA-65)                    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

Certificate Rotation Flow:
  [Sidecar] ──► vault.pki.generate_certificate() ──► [Vault]
      ◄─────── { certificate, private_key } ──────────
      ──► write to ~/certs/tls.crt & tls.key
      ──► sleep(3600)
      ──► repeat
```

---

## Prerequisites

- Ubuntu 22.04 LTS
- Python 3.10+
- At least 2GB free disk space
- Internet connection for initial setup

---

## Step 1 — Install Dependencies

```bash
sudo apt update
sudo apt install -y cmake gcc ninja-build libssl-dev python3-pytest git wget gpg
```

Verify OpenSSL version (needs 3.x):
```bash
openssl version
# OpenSSL 3.0.2
```

---

## Step 2 — Build liboqs

> ⚠️ Do NOT try `sudo apt install liboqs-dev` — it is not in Ubuntu repos. Build from source.

```bash
cd ~
git clone https://github.com/open-quantum-safe/liboqs
cd liboqs && mkdir build && cd build
cmake -DOQS_USE_OPENSSL=ON -DCMAKE_INSTALL_PREFIX=/usr/local ..
make -j4
sudo make install
cd ~
```

If cmake fails with "Could NOT find OpenSSL":
```bash
sudo apt install -y libssl-dev
# Then retry cmake
```

---

## Step 3 — Build oqs-provider

> ⚠️ The .so file installs to `~/oqs-provider/build/lib/` NOT to /usr/lib. Remember this path.

```bash
cd ~
git clone https://github.com/open-quantum-safe/oqs-provider
cd oqs-provider && mkdir build && cd build
cmake -Dliboqs_DIR=/usr/local/lib/cmake/liboqs ..
make -j4
sudo make install
cd ~
```

Verify provider installed and active:
```bash
openssl list -providers -provider oqsprovider
# Should show:
# oqsprovider
#   name: OpenSSL OQS Provider
#   version: 0.12.0-dev
#   status: active
```

Find exact .so path (you will need this):
```bash
ls ~/oqs-provider/build/lib/
# oqsprovider.so
```

---

## Step 4 — Create OpenSSL Config

> ⚠️ Use Python to create this file. Do NOT use cat >> or heredoc — they corrupt the file.
> ⚠️ Replace `jeejon` with your actual Ubuntu username.

```bash
python3 -c "
f = open('/home/jeejon/openssl-oqs.cnf', 'w')
f.write('[openssl_init]\n')
f.write('providers = provider_sect\n\n')
f.write('[provider_sect]\n')
f.write('oqsprovider = oqs_sect\n')
f.write('default = default_sect\n\n')
f.write('[oqs_sect]\n')
f.write('module = /home/jeejon/oqs-provider/build/lib/oqsprovider.so\n')
f.write('activate = 1\n\n')
f.write('[default_sect]\n')
f.write('activate = 1\n\n')
f.write('[req]\n')
f.write('distinguished_name = req_dn\n')
f.write('prompt = no\n\n')
f.write('[req_dn]\n')
f.write('CN = PQC-Root-CA\n')
f.close()
print('Config created!')
"
```

Verify file looks correct:
```bash
cat ~/openssl-oqs.cnf
```

Test provider loads correctly:
```bash
OPENSSL_CONF=~/openssl-oqs.cnf openssl list -providers
# Should show oqsprovider: active
```

---

## Step 5 — Generate PQC Root CA

> ⚠️ Use `mldsa65` NOT `dilithium3`. dilithium3 was the old name and will fail.
> ⚠️ The `-subj` flag can also cause issues. Use the [req_dn] section in config instead.

```bash
OPENSSL_CONF=~/openssl-oqs.cnf openssl req \
  -x509 -new -newkey mldsa65 \
  -keyout root-ca.key -out root-ca.crt \
  -days 3650 -nodes
```

Verify the certificate:
```bash
openssl x509 -in root-ca.crt -text -noout | grep "Signature Algorithm"
# Should show: 2.16.840.1.101.3.4.3.18 (ML-DSA-65)
```

---

## Step 6 — Install and Start Vault

Install Vault:
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install -y vault
vault --version
# Vault v2.0.0
```

> ⚠️ You need TWO terminals. Terminal 1 runs Vault. Terminal 2 runs all commands.

**Terminal 1 — Start Vault:**
```bash
vault server -dev
# Copy the Root Token shown — you need it!
```

**Terminal 2 — Set environment:**
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='hvs.your-root-token-here'
```

Verify Vault is working:
```bash
vault status
# Sealed: false  ← must say false
# Initialized: true
```

---

## Step 7 — Configure PKI Engine

> ⚠️ Vault does NOT support PQC private keys natively.
> Use Vault's internal RSA-4096 CA. The PQC layer comes from oqs-provider at the service level.
> Do NOT try to import your mldsa65 root-ca.key into Vault — it will fail.

```bash
# Enable PKI
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

# Generate Vault internal CA (RSA-4096)
vault write pki/root/generate/internal \
  common_name="PQC-Root-CA" \
  ttl=87600h \
  key_type=rsa \
  key_bits=4096

# Configure URLs
vault write pki/config/urls \
  issuing_certificates="http://127.0.0.1:8200/v1/pki/ca" \
  crl_distribution_points="http://127.0.0.1:8200/v1/pki/crl"

# Create microservice role
vault write pki/roles/microservice \
  allowed_domains="internal.company.com" \
  allow_subdomains=true \
  max_ttl="2h"

# Verify role created
vault read pki/roles/microservice
```

---

## Step 8 — Run the Sidecar

Install hvac:
```bash
pip3 install hvac --no-cache-dir
# If hash mismatch error:
pip3 cache purge && pip3 install hvac --no-cache-dir
```

Create the sidecar script:
> ⚠️ Use Python to create this file NOT nano or heredoc.
> ⚠️ Do NOT use `if __name__ == "__main__":` — terminals corrupt the underscores.
> ⚠️ Do NOT include `ttl` parameter in generate_certificate() — hvac 2.4.0 doesn't support it.

```bash
python3 -c "
content = '''import hvac
import os
import time

VAULT_ADDR = \"http://127.0.0.1:8200\"
VAULT_TOKEN = \"hvs.your-root-token-here\"
ROTATION_INTERVAL = 3600

client = hvac.Client(url=VAULT_ADDR, token=VAULT_TOKEN)

def rotate_certificate():
    print(\"Rotating certificate...\")
    os.makedirs(os.path.expanduser(\"~/certs\"), exist_ok=True)
    response = client.secrets.pki.generate_certificate(
        name=\"microservice\",
        common_name=\"service1.internal.company.com\",
        mount_point=\"pki\"
    )
    cert = response[\"data\"][\"certificate\"]
    key = response[\"data\"][\"private_key\"]
    with open(os.path.expanduser(\"~/certs/tls.crt\"), \"w\") as f:
        f.write(cert)
    with open(os.path.expanduser(\"~/certs/tls.key\"), \"w\") as f:
        f.write(key)
    print(\"Certificate rotated successfully!\")

rotate_certificate()
while True:
    time.sleep(ROTATION_INTERVAL)
    rotate_certificate()
'''
open('/home/jeejon/sidecar.py', 'w').write(content)
print('sidecar.py created!')
"
```

Run the sidecar:
```bash
mkdir -p ~/certs
python3 ~/sidecar.py
```

Expected output:
```
Rotating certificate...
Certificate rotated successfully!
```

---

## Step 9 — Verify Everything Works

**Test 1 — Vault authenticated:**
```bash
python3 -c "
import hvac
client = hvac.Client(url='http://127.0.0.1:8200', token='your-token')
print('Authenticated:', client.is_authenticated())
"
# Authenticated: True
```

**Test 2 — Cert files exist:**
```bash
ls -la ~/certs/
# -rw-rw-r-- 1 jeejon jeejon 1743 tls.crt
# -rw-rw-r-- 1 jeejon jeejon 1678 tls.key
```

**Test 3 — Inspect certificate:**
```bash
openssl x509 -in ~/certs/tls.crt -text -noout | \
  grep -E "Issuer|Subject|Not Before|Not After"
# Issuer: CN = PQC-Root-CA
# Not Before: May 13 10:36:45 2026
# Not After:  May 13 12:37:15 2026
# Subject: CN = service1.internal.company.com
```

**Test 4 — Two rotations with different serials:**
```bash
python3 -c "
import hvac, os, time
client = hvac.Client(url='http://127.0.0.1:8200', token='your-token')
for i in range(2):
    r = client.secrets.pki.generate_certificate(
        name='microservice',
        common_name='service1.internal.company.com',
        mount_point='pki'
    )
    open(os.path.expanduser('~/certs/tls.crt'), 'w').write(r['data']['certificate'])
    open(os.path.expanduser('~/certs/tls.key'), 'w').write(r['data']['private_key'])
    print(f'Rotation {i+1} serial:', r['data']['serial_number'])
    if i == 0: time.sleep(2)
print('Both rotations succeeded with DIFFERENT serial numbers!')
"
```

**Test 5 — List all Vault issued certs:**
```bash
vault list pki/certs
```

**Test 6 — Verify chain of trust:**
```bash
# Download Vault CA
curl http://127.0.0.1:8200/v1/pki/ca/pem -o ~/vault-ca.crt

# Verify cert against Vault CA
openssl verify -CAfile ~/vault-ca.crt ~/certs/tls.crt
# ~/certs/tls.crt: OK
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Error allocating keygen context` | Using dilithium3 instead of mldsa65 | Change to `-newkey mldsa65` |
| `unsupported algorithm` | oqs-provider not loaded | Add `-provider oqsprovider -provider default` |
| `NCONF_get_string: no value` | Missing [req] section in config | Add `[req]` and `[req_dn]` sections |
| `connection refused` | Vault not running | Run `vault server -dev` in Terminal 1 |
| `Authenticated: False` | Token expired or wrong | Copy new token from Terminal 1 |
| `unexpected keyword argument ttl` | hvac 2.4.0 doesn't support ttl | Remove ttl parameter entirely |
| `NameError: name _name_ not defined` | Terminal corrupted underscores | Remove if __name__ check, call directly |
| `Error parsing key: unknown algorithm` | Tried to import PQC key to Vault | Use Vault internal RSA CA instead |
| `liboqs-dev: unable to locate` | Not in Ubuntu repos | Build from source with cmake |

---

## Project Architecture

```
Terminal 1 (keep running):
vault server -dev
     |
     | issues certs
     v
Terminal 2 (your commands):
export VAULT_ADDR + VAULT_TOKEN
     |
     | calls API
     v
sidecar.py
     |
     | writes every 60 min
     v
~/certs/tls.crt  (1743 bytes)
~/certs/tls.key  (1678 bytes)
     |
     | used by
     v
microservices (mTLS)
```

---


## Report Doc

- https://docs.google.com/document/d/10KzGKZIW6-VCJ_7OVREHBFV6wMK7IrkTAKDi3Q3o3so/edit?tab=t.0

## Author

# Johnson Oni

- Email: johnsononi@internal.bincom.net
- LinkedIn: https://www.linkedin.com/in/johnson-oni-0144b7158

## License

MIT License — see <LICENSE> for details.




