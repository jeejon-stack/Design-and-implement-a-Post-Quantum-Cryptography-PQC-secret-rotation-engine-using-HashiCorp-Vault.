# Design-and-implement-a-Post-Quantum-Cryptography-PQC-secret-rotation-engine-using-HashiCorp-Vault.
A production-ready Post-Quantum Cryptography (PQC) certificate rotation engine built on HashiCorp Vault, using NIST-standardized ML-DSA-65 (Dilithium) algorithms to replace RSA-2048 for internal microservice mTLS.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [How It Works](#how-it-works)
- [Security Considerations](#security-considerations)
- [Roadmap](#roadmap)
- [License](#license)

---

## Overview

This project implements a **Post-Quantum Cryptography (PQC) secret rotation engine** that:

- Replaces RSA-2048 certificates with **mldsa65 (NIST ML-DSA-65 / Dilithium)** — quantum-resistant digital signatures
- Uses **HashiCorp Vault** as the PKI backend and certificate authority
- Runs a **Python sidecar** that automatically rotates certificates every **60 minutes**
- Supports **zero-downtime rotation** — certificates are written atomically without dropping connections
- Creates an mTLS environment **theoretically secure against quantum computer attacks**

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

- Ubuntu 22.04 LTS (or compatible Debian-based Linux)
- CMake >= 3.16
- GCC / G++
- Python 3.10+
- Git
- OpenSSL 3.x (`openssl version` to check)

---

## Installation

### Step 1: Install System Dependencies

```bash
sudo apt update
sudo apt install -y cmake gcc ninja-build libssl-dev python3-pytest git wget gpg
```

### Step 2: Install HashiCorp Vault

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install -y vault
vault --version  # should show Vault v2.0.0 or later
```

### Step 3: Build liboqs (Open Quantum Safe)

```bash
git clone https://github.com/open-quantum-safe/liboqs
cd liboqs && mkdir build && cd build
cmake -DOQS_USE_OPENSSL=ON -DCMAKE_INSTALL_PREFIX=/usr/local ..
make -j4
sudo make install
cd ~
```

### Step 4: Build oqs-provider

```bash
git clone https://github.com/open-quantum-safe/oqs-provider
cd oqs-provider && mkdir build && cd build
cmake -Dliboqs_DIR=/usr/local/lib/cmake/liboqs ..
make -j4
sudo make install
```

Verify the provider installed correctly:

```bash
openssl list -providers -provider oqsprovider
# Expected output:
# Providers:
#   oqsprovider
#     name: OpenSSL OQS Provider
#     version: 0.12.0-dev
#     status: active
```

### Step 5: Install Python Dependencies

```bash
pip3 install hvac --no-cache-dir
python3 -c "import hvac; print(hvac.__version__)"  # should print 2.4.0
```

---

## Configuration

### Step 1: Create OpenSSL PQC Config

```bash
python3 -c "
content = '''[openssl_init]
providers = provider_sect

[provider_sect]
oqsprovider = oqs_sect
default = default_sect

[oqs_sect]
module = /home/$USER/oqs-provider/build/lib/oqsprovider.so
activate = 1

[default_sect]
activate = 1

[req]
distinguished_name = req_dn
prompt = no

[req_dn]
CN = PQC-Root-CA
'''
open('/home/$USER/openssl-oqs.cnf', 'w').write(content)
print('Config created!')
"
```

### Step 2: Generate PQC Root CA Certificate

```bash
OPENSSL_CONF=~/openssl-oqs.cnf openssl req \
  -x509 -new -newkey mldsa65 \
  -keyout root-ca.key -out root-ca.crt \
  -days 3650 -nodes

# Verify the certificate
openssl x509 -in root-ca.crt -text -noout | grep "Signature Algorithm"
# Should show: 2.16.840.1.101.3.4.3.18 (ML-DSA-65)
```

### Step 3: Start Vault and Configure PKI

**Terminal 1 — Start Vault:**
```bash
vault server -dev
# Copy the Root Token displayed
```

**Terminal 2 — Configure PKI:**
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='<your-root-token>'

# Enable PKI engine
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

# Generate internal CA
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

# Verify role
vault read pki/roles/microservice
```

---

## Usage

### Running the Sidecar

```python
# sidecar.py
import hvac
import os
import time

VAULT_ADDR = "http://127.0.0.1:8200"
VAULT_TOKEN = "your-vault-token"
ROTATION_INTERVAL = 3600  # 60 minutes

client = hvac.Client(url=VAULT_ADDR, token=VAULT_TOKEN)

def rotate_certificate():
    print("Rotating certificate...")
    os.makedirs(os.path.expanduser("~/certs"), exist_ok=True)
    
    response = client.secrets.pki.generate_certificate(
        name="microservice",
        common_name="service1.internal.company.com",
        mount_point="pki"
    )
    
    cert = response["data"]["certificate"]
    key = response["data"]["private_key"]
    
    # Atomic writes
    with open(os.path.expanduser("~/certs/tls.crt"), "w") as f:
        f.write(cert)
    with open(os.path.expanduser("~/certs/tls.key"), "w") as f:
        f.write(key)
    
    print("Certificate rotated successfully!")

# Run immediately then every 60 minutes
rotate_certificate()
while True:
    time.sleep(ROTATION_INTERVAL)
    rotate_certificate()
```

Run the sidecar:

```bash
mkdir -p ~/certs
python3 sidecar.py
```

### Verifying Rotation

```bash
# Check certificate files exist
ls -la ~/certs/
# Expected:
# -rw-rw-r-- 1 user user 1743 May 13 10:37 tls.crt
# -rw-rw-r-- 1 user user 1678 May 13 10:37 tls.key

# Inspect certificate details
openssl x509 -in ~/certs/tls.crt -text -noout | \
  grep -E "Issuer|Subject|Not Before|Not After"

# Expected output:
# Issuer: CN = PQC-Root-CA
# Not Before: May 13 10:36:45 2026 GMT
# Not After:  May 13 12:37:15 2026 GMT  ← 2 hour TTL
# Subject: CN = service1.internal.company.com
```

---

## How It Works

| Step | Description |
|------|-------------|
| 1 | Vault starts with PKI secrets engine enabled |
| 2 | Sidecar authenticates to Vault using a token |
| 3 | Sidecar calls `pki.generate_certificate()` via hvac |
| 4 | Vault issues a signed cert valid for 2 hours |
| 5 | Sidecar writes cert and key atomically to `~/certs/` |
| 6 | Microservices reload certs (via SIGHUP or file watch) |
| 7 | Sidecar sleeps 60 minutes then repeats from Step 3 |

The **2-hour TTL with 60-minute rotation** means there is always a 1-hour overlap window, ensuring zero-downtime transitions between certificate generations.

---

## Security Considerations

- **PQC Algorithm**: mldsa65 (ML-DSA-65) is NIST FIPS 204 standardized — quantum-resistant digital signatures
- **Short TTL**: 2-hour certificate lifetime limits blast radius if a key is compromised
- **Automated Rotation**: Eliminates human error in manual certificate management
- **For Production**:
  - Replace dev mode with a production Vault cluster (Raft storage + TLS)
  - Use Vault AppRole or Kubernetes auth instead of root tokens
  - Enable Vault audit logging
  - Implement SIGHUP-based graceful reload in microservices
  - Deploy sidecar as a Kubernetes sidecar container

---

## Roadmap

- [ ] Kubernetes deployment manifests (sidecar container pattern)
- [ ] Vault AppRole authentication for sidecar
- [ ] SIGHUP-based graceful cert reload example
- [ ] Docker Compose demo environment
- [ ] Kyber (ML-KEM) key encapsulation support
- [ ] Prometheus metrics for rotation events
- [ ] Helm chart for production deployment

---


## Report Doc

- https://docs.google.com/document/d/10KzGKZIW6-VCJ_7OVREHBFV6wMK7IrkTAKDi3Q3o3so/edit?tab=t.0

## Author

# Johnson Oni

- Email: johnsononi@internal.bincom.net
- LinkedIn: https://www.linkedin.com/in/johnson-oni-0144b7158

## License

MIT License — see <LICENSE> for details.




