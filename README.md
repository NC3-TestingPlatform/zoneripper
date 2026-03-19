# ZoneRipper

**ZoneRipper** is a Python tool for assessing **DNSSEC zone-walking exposure**.

It automatically detects whether a DNS zone uses **NSEC** or **NSEC3**, performs **NSEC zone enumeration**, and can **actively collect NSEC3 hashes** for dictionary cracking or GPU cracking using **Hashcat**.

---

# Features

## DNSSEC Detection

ZoneRipper first checks whether DNSSEC is enabled by querying for **DNSKEY records** at the zone apex.

If DNSSEC is not enabled, zone walking is not possible.

---

## NSEC Zone Walking

If a zone uses **plain NSEC**, the tool:

* Detects the NSEC denial-of-existence mechanism
* Follows the **NSEC chain**
* Enumerates all discoverable owner names
* Extracts **RR types present at each hostname**
* Detects and skips **trap / synthetic names** used by hardened nameservers

Plain NSEC allows **full zone enumeration**.

---

## NSEC3 Hash Collection

For zones using **NSEC3**, ZoneRipper performs an **active gap-targeted hash walk**:

1. Bootstrap the hash ring using a deterministic probe
2. Track discovered hash intervals
3. Identify **uncovered gaps in the hash space**
4. Generate candidate labels whose hashes fall into those gaps
5. Query DNS to collect additional NSEC3 records

This technique efficiently reconstructs the **entire NSEC3 hash ring**.

Collected data includes:

* hashing algorithm
* iteration count
* salt
* owner hash
* next hash
* record types

---

## NSEC3 Hash Cracking

ZoneRipper includes a **pure Python NSEC3 hashing implementation** based on RFC 5155.

It supports dictionary attacks to recover plaintext subdomain labels.

Features:

* automatic DNS label validation
* iteration-aware hashing
* salt handling
* cracked subdomain reporting

Example output:

```
CRACKED: 9CLK...FOJ → admin.example.com
```

---

## Hashcat Export

For large datasets, hashes can be exported for **GPU cracking**.

Export format follows **Hashcat mode 8300**:

```
<hash>:<.zone>:<salt>:<iterations>
```

Example:

```
9clkef9t1cpn5jp5ltaohtp49dqi9foj:.example.com:73:0
```

Crack using:

```
hashcat -m 8300 example.com_nsec3.hashes wordlist.txt --keep-guessing
```

---

## Robust Resolver Logic

ZoneRipper improves reliability by:

* resolving **all authoritative nameservers**
* rotating queries across nameservers
* handling **anycast DNS infrastructures**
* retrying unresponsive nodes

---

# Installation

Requirements:

* Python **3.8+**

Install dependency:

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

```
python3 zoneripper.py example.com
```

The tool will:

1. Detect DNSSEC
2. Identify NSEC/NSEC3
3. Attempt enumeration or hash collection

---

## Limit NSEC Walk Steps

Limit the number of NSEC hops:

```
python3 zoneripper.py example.com --max-steps 100
```

Use `0` for unlimited.

---

## Specify a Nameserver

Use a specific resolver or authoritative server:

```
python3 zoneripper.py example.com --nameserver 8.8.8.8
```

---

## Increase NSEC3 Collection Rounds

```
python3 zoneripper.py example.com --nsec3-rounds 200
```

Use `0` for unlimited rounds.

---

## Crack NSEC3 Hashes

```
python3 zoneripper.py example.com --wordlist rockyou.txt
```

Uncracked hashes will be exported to:

```
example.com_nsec3.hashes
```

---

# Python API Usage

ZoneRipper can also be used as a **Python library**.

Example:

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

```
{
    "nsec_type": "NSEC | NSEC3 | NONE | UNKNOWN",
    "nsec_names": [...],
    "nsec3": Nsec3WalkResult
}
```

---

# Security Implications

## NSEC

Plain **NSEC** exposes the full zone contents.

Attackers may discover:

* internal services
* staging environments
* forgotten subdomains
* hidden infrastructure

Recommended mitigation:

```
Use NSEC3 with opt-out
```

---

## NSEC3

NSEC3 reduces enumeration but may still leak names if:

* weak dictionary words are used
* iteration count is low
* predictable subdomains exist

---

# Use Cases

ZoneRipper is useful for:

* DNS security assessments
* penetration testing
* red team reconnaissance
* bug bounty research
* DNSSEC deployment verification

---

# Limitations

* NSEC3 cracking is **CPU-only in Python**
* Large hash sets should be exported to **Hashcat**
* Only **SHA-1 NSEC3 (RFC 5155)** is supported
* Very large zones may require many collection rounds

---

## License

GPLv3 — see [LICENSE](LICENSE) for details.