# CTF OSINT Writeup: 01 CERT UP
<img width="490" height="432" alt="image" src="https://github.com/user-attachments/assets/40b5eb3d-0929-498e-bb05-097ff2dc3eb1" />

> **Category:** OSINT | **Platform:** Outback Overflow '26   
---

## Challenge 01: CERT UP

### 1. Challenge Overview
* **Category:** OSINT (Public Key Infrastructure & Certificate Transparency)
* **Points:** 100 Points
* **Challenge Prompt:**  
  > *"To be at peace, one needs to be transparent."*  
  > *Flag format:* `OO{...}`
* **Flag Captured:** `OO{i-am-at-p3ace}`

---

### 2. Clue Deconstruction & Technical Methodology
The challenge description contains two critical linguistic and architectural clues:
1. **"transparent"**: Refers directly to **Certificate Transparency (CT)** (RFC 6962). CT is an open, append-only, cryptographically verified framework of public ledgers monitoring every SSL/TLS certificate issued by publicly trusted Certificate Authorities (CAs).
2. **"at peace"**: Serves as the target semantic anchor for the subdomain hostname created for the challenge.

When certificates are issued (e.g., via Let's Encrypt or ZeroSSL), CAs are mandated to submit pre-certificates and certificates to CT logs (such as Google Argon, Cloudflare Nimbus, or DigiCert). Security analysts and OSINT investigators query these logs via public search aggregators like **crt.sh** or **Censys** to uncover hidden infrastructure, subdomains, and staging environments.

---

### 3. Step-by-Step Investigation Walkthrough

#### Step 1: Querying the Public Certificate Transparency Ledger
The primary domain for the competition platform was `hscyber.org`. To discover all issued certificates across any subdomains (including wildcards), we formulate a wildcard search query against `crt.sh`:

```bash
# Query crt.sh JSON API for all certificates registered under *.hscyber.org
curl -s "https://crt.sh/?q=%25.hscyber.org&output=json" | jq .
```
<img width="655" height="358" alt="image" src="https://github.com/user-attachments/assets/f0f94569-fa7e-4586-9bf0-2477d60896b8" />

#### Step 2: Isolating the Target Certificate Record
Filtering the query results for certificates containing the keyword `"peace"` or auditing recently provisioned certificates revealed a ZeroSSL certificate issued on **October 6, 2026**:

* **Issuer:** ZeroSSL ECC Domain Secure Site CA
* **Serial Number:** `7b2868146f439536eb73bc3646dae256`
* **Entry Timestamp:** `2026-10-06T14:22:18Z`
* **Common Name (CN):** `i-am-at-p3ace.hscyber.org`
* **Subject Alternative Name (SAN):** `DNS:i-am-at-p3ace.hscyber.org`

#### Step 3: DNS Resolution & Decoy Verification
We tested whether `i-am-at-p3ace.hscyber.org` was backed by an active web server or existed solely as an OSINT breadcrumb in the CT ledger:

```bash
dig +noall +answer i-am-at-p3ace.hscyber.org
# Result: NXDOMAIN (Non-Existent Domain)
```

The DNS query returned `NXDOMAIN`, confirming that no A/AAAA records exist. The certificate was intentionally generated solely to be ingested into public CT log monitors.

#### Step 4: Flag Formulation
Following the specified format `OO{...}` and wrapping the discovered subdomain slug (`i-am-at-p3ace`):
``` Bash
Flag= OO{i-am-at-p3ace}
```
* **Evidence File:** [`evidence/cert_up_transparency_log.txt`](evidence/cert_up_transparency_log.txt)

---

