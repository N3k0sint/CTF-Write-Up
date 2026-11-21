# CTF Writeup: DISGRUNTLEDDEV IV
<img width="597" height="736" alt="image" src="https://github.com/user-attachments/assets/e459c42a-d8d4-4a13-a942-a2d4d9ecd25a" />

| Field | Value |
|---|---|
| **Category** | `Network Forensics / Covert Channels` |
| **Difficulty** | `Hard` |
| **Points** | `100` |
| **Author** | `digitalsurf` |
| **Target** | `Marcus (192.168.1.47)` |
| **PCAP File** | `dd4_insane.pcap` |
| **Final Flag** | `OO{ph4nt0m_pr0t0c0l_m4st3r_cl4ss}` |

---

## EXECUTIVE SUMMARY

Marcus has abandoned standard payload-based exfiltration. Instead, he created a 
custom covert channel named "Operation Phantom." Standard DLP and IDS systems 
missed this because the data was hidden entirely within the packet headers 
(DNS Transaction IDs and IP Identification fields). 

To crack this, we had to locate the blueprint, write a Scapy script to extract 
the raw header fields, realize that the high bytes were randomized noise, mask 
them out using bitwise AND (& 0xFF), and finally reverse the XOR -> Base64 
encoding to reveal the flag.

## PHASE 1: THE BLUEPRINT AND INITIAL RECON

The challenge description revealed that Marcus left the blueprint inside an 
HTML comment on an internal wiki page. 

**Target:** `GET /wiki/Project_Phantom/Architecture (wiki.initech.local)`

### Blueprint Found

```html
<!-- TODO cleanup: phantom debug config
     protocol_map:
       key_channel:  dns_txid  @ resolver 198.51.100.200
       data_channel: ip_id     @ echo 203.0.113.200
       encoding:     xor -> base64
       sort:         icmp_seq asc
-->
```

### Analysis of the Blueprint

1. Key Channel: DNS Transaction IDs (dns.id) sent to 198.51.100.200.
2. Data Channel: IP Identification (ip.id) of ICMP Echo Requests sent to 203.0.113.200.
3. Encoding: The data is XORed, then Base64 encoded.
4. Sorting: The ICMP packets must be sorted by their ICMP sequence number (icmp.seq) in ascending order.

## PHASE 2: THE FIRST ATTEMPT (AND WHY IT FAILED)

We wrote a Python script using the Scapy library to extract the full 2-byte 
(16-bit) dns.id and ip.id fields, sort them, XOR them, and Base64 decode them.

### Initial Script (Failed)

```python
import base64
from scapy.all import rdpcap, IP, UDP, DNS, ICMP

pkts = rdpcap("dd4_insane.pcap")

# 1. Extract key from DNS Transaction IDs
key_bytes = b""
for p in pkts:
    if p.haslayer(IP) and p.haslayer(DNS):
        if p[IP].src == "192.168.1.47" and p[IP].dst == "198.51.100.200" and p[DNS].qr == 0:
            key_bytes += p[DNS].id.to_bytes(2, "big")

# 2. Extract data from IP IDs of ICMP Echo Requests
icmp_chunks = []
for p in pkts:
    if p.haslayer(IP) and p.haslayer(ICMP):
        if p[IP].src == "192.168.1.47" and p[IP].dst == "203.0.113.200" and p[ICMP].type == 8:
            icmp_chunks.append((p[ICMP].seq, p[IP].id.to_bytes(2, "big")))

icmp_chunks.sort(key=lambda x: x[0])
data_bytes = b"".join(chunk for _, chunk in icmp_chunks)

# 3. Reverse encoding
xored = bytes(b ^ key_bytes[i % len(key_bytes)] for i, b in enumerate(data_bytes))
print("[xor -> base64]:", base64.b64decode(xored))
```

### Output

```text
[base64 -> xor] failed: Incorrect padding
[xor -> base64]: b'OO{\xed\xc1\xa0\xd1f\xe7\x99\rpm_pr0t0o\x0c\xc1\xb1p\xf4\xfbL5\x11\xcd\xd2\xb37%\xf6>\xdb\r\x1c\x13s}'

Why did this fail?
The string started with "OO{" but immediately turned into garbage bytes. We 
realized that the DNS Transaction ID and IP ID fields are 16-bit (2-byte) 
integers. Marcus was putting our actual data into the LOW byte, and filling 
the HIGH byte with random noise to evade IDS signature detection.
```

## PHASE 3: THE BREAKTHROUGH (BITMASKING WITH & 0xFF)

To extract only the meaningful low byte and discard the random high byte, we 
used a bitwise AND operation: `value & 0xFF`. 

This masks out everything except the last 8 bits (the low byte).

We updated the script to apply `& 0xFF` to both the DNS IDs and the IP IDs.

### Final Script
<img width="715" height="625" alt="image" src="https://github.com/user-attachments/assets/27b2e991-a5cb-4f2a-aaeb-4401c18725d0" />

```python
import base64
from scapy.all import rdpcap, IP, DNS, ICMP

pkts = rdpcap("dd4_insane.pcap")

# 1. Extract LOW BYTE (& 0xFF) of DNS Transaction IDs -> key
key = bytes(
    p[DNS].id & 0xFF
    for p in pkts
    if p.haslayer(IP) and p.haslayer(DNS)
    and p[IP].src == "192.168.1.47"
    and p[IP].dst == "198.51.100.200"
    and p[DNS].qr == 0
)

# 2. Extract LOW BYTE (& 0xFF) of IP IDs from ICMP Echo Requests, sorted by icmp_seq
icmp_chunks = [
    (p[ICMP].seq, p[IP].id & 0xFF)
    for p in pkts
    if p.haslayer(IP) and p.haslayer(ICMP)
    and p[IP].src == "192.168.1.47"
    and p[IP].dst == "203.0.113.200"
    and p[ICMP].type == 8
]
icmp_chunks.sort(key=lambda x: x[0])
data = bytes(b for _, b in icmp_chunks)

# 3. XOR with key and Base64-decode
xored = bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
print("Key:", key.decode())
print("Base64:", xored.decode())
print("Flag:", base64.b64decode(xored).decode())
```

## PHASE 4: EXECUTION AND VERIFICATION

We ran the updated script against the PCAP file.

### Command

```bash
$ python3 extract.py
```

### Output

```text
Key: Op3r4t10nPh4nt0m
Base64: T097cGg0bnQwbV9wcjB0MGMwbF9tNHN0M3JfY2w0c3N9
Flag: OO{ph4nt0m_pr0t0c0l_m4st3r_cl4ss}

The output provided the clean flag, matching the required OO{...} format.

Flag Submitted: OO{ph4nt0m_pr0t0c0l_m4st3r_cl4ss}
Result: CORRECT
```

## KEY TAKEAWAYS & LESSONS LEARNED

1. Covert Channels: Data exfiltration doesn't require payloads. Attackers can 
   hide data in IP IDs, TCP sequence numbers, DNS Transaction IDs, or ICMP 
   timestamps. Always inspect header fields.
2. The "Noise" Defense: High-entropy randomization of unused header bytes is a 
   common evasion tactic. When extracting data from 16-bit fields, always check 
   if masking the high byte (& 0xFF) reveals a pattern.
3. Chaining Encodings: Always pay attention to the exact order of operations. 
   The blueprint specified "xor -> base64", meaning you must XOR the raw bytes 
   first, and then Base64 decode the result.
4. Tooling: Scapy is essential for parsing PCAPs when standard tools like 
   tshark fall short. Accessing raw layer fields (p[DNS].id, p[IP].id) allows 
   for precise data carving.
