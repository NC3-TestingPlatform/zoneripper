# ZoneRipper

A Python tool for testing **DNSSEC zone walking vulnerabilities**.

This tool detects whether a domain uses **NSEC** or **NSEC3**, attempts **NSEC zone walking**, and can **collect and crack NSEC3 hashes** using dictionary attacks or export them for **GPU cracking with Hashcat**.

---

## Features

### DNSSEC Detection
- Checks if DNSSEC is enabled via DNSKEY records.

### NSEC Zone Walking
- Detects **plain NSEC**
- Automatically walks the **entire NSEC chain**
- Enumerates **all discoverable hostnames**
- Detects and skips **trap / synthetic names**

### NSEC3 Hash Collection
- Collects hashes via randomized probes
- Extracts:
  - algorithm
  - iterations
  - salt
  - owner hash
  - next hash

### NSEC3 Hash Cracking
- Built-in **pure Python SHA-1 implementation**
- Dictionary attack support
- Automatically skips invalid DNS labels
- Reports cracked subdomains

### Hashcat Export
Exports hashes in **Hashcat mode 8300 format**:

```
<hash>:<.zone>:<salt>:<iterations>
```

Example:

```
9clkef9t1cpn5jp5ltaohtp49dqi9foj:.example.com:73:0
```

Use with:

```
hashcat -m 8300 hashes.txt wordlist.txt --keep-guessing
```

### Robust Resolver Logic
- Queries **all authoritative nameservers**
- Handles **anycast DNS infrastructure**
- Automatically retries failed nodes

---

# Installation

Requires Python **3.8+**

```bash
pip install dnspython
```

Clone the repository:

```bash
git clone https://github.com/t0kubetsu/zoneripper.git
cd zoneripper
```

---

# Usage

## Basic Scan

```bash
python3 zoneripper.py example.com
```

---

## Limit NSEC Walk Steps

```bash
python3 zoneripper.py example.com --max-steps 100
```

---

## Use a Specific Resolver

```bash
python3 zoneripper.py example.com --nameserver 8.8.8.8
```

---

## Collect More NSEC3 Hashes

```bash
python3 zoneripper.py example.com --nsec3-rounds 100
```

---

## Crack NSEC3 Hashes With a Wordlist

```bash
python3 zoneripper.py example.com --wordlist rockyou.txt
```

Uncracked hashes will be exported to:

```
example.com_nsec3.hashes
```

---

# Python API Usage

The tool can also be used as a **library**.

```python
from zoneripper import run

results = run(
    "example.com",
    max_steps=100,
    nameserver="8.8.8.8"
)

print(results)
```

Returned structure:

```json
{
    "nsec_type": "NSEC | NSEC3 | NONE | UNKNOWN",
    "nsec_names": [...],
    "nsec3": Nsec3WalkResult
}
```

---

# Security Implications

### NSEC

Plain **NSEC** allows full zone enumeration.

Attackers can discover:

* internal hosts
* staging environments
* forgotten subdomains

### NSEC3

NSEC3 protects against enumeration but may still leak names if:

* weak wordlists are used
* iteration count is low
* common hostnames exist

---

# Use Cases

* DNS security audits
* penetration testing
* bug bounty research
* DNSSEC configuration verification
* zone enumeration analysis

---

# Limitations

* NSEC3 cracking is **CPU-only** in Python
* Large hash sets should be exported to **Hashcat**
* Only **SHA-1 NSEC3 (RFC 5155)** is supported
