# Hackerverse DFIR CTF

<img width="600" height="800" alt="image" src="https://github.com/user-attachments/assets/ba51011a-8d09-4bd9-aa04-e18f65bf629e" />


A collection of writeups, investigation methodologies, and forensic artifacts from the **Hackerverse Digital Forensics & Incident Response (DFIR) CTF**, hosted by [EC-Council](https://hackerverse.com).

## Overview

The Hackerverse DFIR CTF is a live virtual machine (VM) Capture-the-Flag competition focused on real-world incident investigation and forensic analysis. Participants are placed inside an active target environment to trace attacker activity, recover hidden data, and extract compromised credentials.


| Field | Details |
| :--- | :--- |
| **Platform** | HackerVerse (EC-Council) |
| **Event** | September 2026 DFIR CTF |
| **Focus Areas** | Steganography, Network Forensics, Windows Credential Extraction, Malware/Ransomware Analysis |
| **Environment** | Live Target Virtual Machines (Linux & Windows) |

---

### Competition Format
* **Environment:** 1 live target Virtual Machine (VM)
* **Time Limit:** 6 hours
* **Structure:** 4 progressive missions across 4 difficulty tiers

## Challenge Levels & Investigation Summary

This repository is organized into four progressive levels, moving from file-level artifact recovery to full ransomware reverse engineering:

| Level | Title | Core Focus | Key Artifacts & Tools |
| :--- | :--- | :--- | :--- |
| **Level 1** | [Cryptic Canvas](./Level-1-Cryptic-Canvas.md) | **Steganography & Encoding** — Decoding Morse-encoded binary audio, repairing corrupted JPEG JFIF headers, and reversing multi-layer ciphers (`Base58 → ROT13 → Base64 → XOR`). | `Audio.wav`, `Cat.jpg`, `Encoded_Dump.txt` (`sox`, `multimon-ng`, `steghide`, `xxd`, Python) |
| **Level 2** | [Network Intrusion Forensics](./Level-2-Network-Intrusion-Forensics.md) | **PCAP & Attack Chain Analysis** — Reconstructing unauthorized account creation, hidden HTTP/WebSocket endpoints, and outbound reverse shell traffic on a compromised ChatOps server. | `synixon_breach_capture.pcapng` (`tshark`, Wireshark) |
| **Level 3** | [Windows Credential Forensics](./Level-3-Windows-Credential-Forensics.md) | **LSASS & DPAPI Decryption** — Extracting DPAPI masterkeys from memory and decrypting Chrome's AES key (`Local State`) to recover saved credentials from `Login Data`. | `lsass.DMP`, `Google.zip` (`Mimikatz`, `Pypykatz`, `Invoke-PowerChrome`, Python) |
| **Level 4** | [Ransomware Analysis & Decryption](./Level-4-Ransomware-Analysis.md) | **Malware Unpacking & Crypto** — Unpacking an MPRESS-compressed `.NET` binary in memory, reversing its machine-name MD5 key derivation, and decrypting AES-128 locked files. | `Rasome.exe`, `Flag.txt.locked` (`Detect It Easy`, `dnSpy`, PowerShell, Python) |

## Repository Structure

```text
├── README.md                                  # Event overview and challenge index
├── Level-1-Cryptic-Canvas.md                  # Steganography, header repair & multi-stage decoding
├── Level-2-Network-Intrusion-Forensics.md     # PCAP triage & ChatOps breach reconstruction
├── Level-3-Windows-Credential-Forensics.md    # LSASS dump analysis & Chrome DPAPI decryption
└── Level-4-Ransomware-Analysis.md             # MPRESS .NET unpacking & AES-128 file recovery
```

## Disclaimer

All writeups, scripts, and methodologies documented in this repository are strictly for educational purposes, defensive research, and authorized CTF practice.
