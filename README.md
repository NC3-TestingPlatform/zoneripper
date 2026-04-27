# zoneripper

> Assess DNSSEC zone-walking exposure — from the command line or as a Python library.

**zoneripper** automatically detects whether a DNS zone uses **NSEC** or **NSEC3**, walks the
NSEC chain to enumerate all owner names, collects NSEC3 hashes via a gap-targeted active walk,
and exports uncracked hashes in **Hashcat mode 8300** format for GPU cracking.

```
$ python3 zoneripper.py example.com
```

![Python](https://img.shields.io/badge/python-%3E%3D3.11-blue)
![License](https://img.shields.io/badge/license-GPLv3-lightgrey)

---

## Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [CLI Usage](#cli-usage)
- [Python API](#python-api)
- [Security Implications](#security-implications)
- [Use Cases](#use-cases)
- [Limitations](#limitations)
- [Contributing](#contributing)

---

## Features

### DNSSEC Detection

zoneripper first checks whether DNSSEC is enabled by querying for **DNSKEY records** at the zone apex.
If DNSSEC is not enabled, zone walking is not possible.

### NSEC Zone Walking

If a zone uses **plain NSEC**, the tool:

* Detects the NSEC denial-of-existence mechanism
* Follows the **NSEC chain**
* Enumerates all discoverable owner names
* Extracts **RR types present at each hostname**
* Detects and skips **trap / synthetic names** used by hardened nameservers

Plain NSEC allows **full zone enumeration**.

### NSEC3 Hash Collection

For zones using **NSEC3**, zoneripper performs an **active gap-targeted hash walk**:

1. Bootstrap the hash ring using a deterministic probe
2. Track discovered hash intervals
3. Identify **uncovered gaps in the hash space**
4. Generate candidate labels whose hashes fall into those gaps
5. Query DNS to collect additional NSEC3 records

Collected data includes the hashing algorithm, iteration count, salt, owner hash, next hash, and record types.

### NSEC3 Hash Cracking

zoneripper includes a **pure Python NSEC3 hashing implementation** based on RFC 5155.
It supports dictionary attacks to recover plaintext subdomain labels, with automatic DNS label
validation, iteration-aware hashing, and salt handling.

```
CRACKED: 9CLK...FOJ → admin.example.com
```

### Hashcat Export

For large datasets, hashes can be exported in **Hashcat mode 8300** format:

```
<hash>:<.zone>:<salt>:<iterations>
```

Example:

```
9clkef9t1cpn5jp5ltaohtp49dqi9foj:.example.com:73:0
```

Crack with:

```bash
hashcat -m 8300 example.com_nsec3.hashes wordlist.txt --keep-guessing
```

### Robust Resolver Logic

* Resolves **all authoritative nameservers** for the zone
* Rotates queries across nameservers
* Handles **anycast DNS infrastructures**
* Retries unresponsive nodes

---

## Requirements

- Python ≥ 3.11
- [`dnspython`](https://www.dnspython.org/) ≥ 2.6

---

## Installation

```bash
git clone https://github.com/NC3-TestingPlatform/zoneripper.git
cd zoneripper
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

---

## CLI Usage

### Basic scan

```bash
python3 zoneripper.py example.com
```

The tool detects DNSSEC, identifies NSEC/NSEC3, then attempts enumeration or hash collection.

### Common options

```bash
# Use a specific resolver or authoritative nameserver
python3 zoneripper.py example.com --nameserver 8.8.8.8

# Limit the number of NSEC walk steps (0 = unlimited)
python3 zoneripper.py example.com --max-steps 100

# Increase NSEC3 collection rounds (0 = unlimited)
python3 zoneripper.py example.com --nsec3-rounds 200

# Dictionary-attack NSEC3 hashes; uncracked hashes exported to example.com_nsec3.hashes
python3 zoneripper.py example.com --wordlist rockyou.txt
```

---

## Python API

```python
from zoneripper import run

results = run(
    "example.com",
    max_steps=100,
    nameserver="8.8.8.8",
)
```

Returned structure:

```python
{
    "nsec_type": "NSEC | NSEC3 | NONE | UNKNOWN",
    "nsec_names": [...],
    "nsec3": Nsec3WalkResult,
}
```

---

## Security Implications

### NSEC

Plain **NSEC** exposes the full zone contents. Attackers may discover internal services,
staging environments, forgotten subdomains, and hidden infrastructure.

Recommended mitigation: use **NSEC3 with opt-out**.

### NSEC3

NSEC3 reduces enumeration but may still leak names if weak dictionary words are used,
the iteration count is low, or predictable subdomains exist.

---

## Use Cases

* DNS security assessments
* Penetration testing
* Red team reconnaissance
* Bug bounty research
* DNSSEC deployment verification

---

## Limitations

* NSEC3 cracking is **CPU-only in Python** — export to Hashcat for large hash sets
* Only **SHA-1 NSEC3 (RFC 5155)** is supported
* Very large zones may require many collection rounds

---

## Contributing

1. Fork the repository and create a feature branch.
2. Test your changes manually against NSEC and NSEC3 zones.
3. Use [conventional commits](https://www.conventionalcommits.org/):
   `fix:`, `feat:`, `refactor:`, `docs:`, `chore:`

---

## License

GPLv3 — see [LICENSE](LICENSE) for details.
