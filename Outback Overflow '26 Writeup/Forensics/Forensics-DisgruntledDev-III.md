# CTF Writeup: DISGRUNTLEDDEV III
<img width="605" height="772" alt="image" src="https://github.com/user-attachments/assets/844aa4d1-16c4-4deb-9c4f-c67469567ae2" />

| Field | Value |
|---|---|
| **Category** | `Network Forensics / PCAP Analysis` |
| **Difficulty** | `Hard` |
| **Points** | `75` |
| **Author** | `digitalsurf` |
| **Target** | `Marcus (192.168.1.47)` |
| **PCAP File** | `dd3_hard.pcap` |
| **Final Flag** | `OO{1cmp_c0v3rt_ch4nn3l_xor_m4st3r}` |

---

## EXECUTIVE SUMMARY

Marcus has escalated his data exfiltration techniques. Instead of simple 
DNS or FTP, he utilized a split-channel exfiltration method:
1. A decoy HTTP POST to an external paste service (paste.hackmd.io) carrying the 
   decryption key hidden in an analytics cookie.
2. ICMP Tunneling to a non-gateway IP (45.33.32.156) containing XOR-encrypted 
   Base64/ASCII payloads.

By analyzing the PCAP, extracting the XOR key from the HTTP cookie, and using 
Scapy to parse the raw ICMP payloads, we successfully decrypted the hidden flag.

## PHASE 1: ISOLATE THE TWO ANOMALOUS STREAMS

The challenge description hinted at two distinct network events. We used tshark 
to pull both streams simultaneously.

### Command 1: Extract HTTP POST requests to hackmd.io

```bash
$ tshark -r dd3_hard.pcap -Y "http.request.method == \"POST\" && http.host contains \"hackmd.io\"" -T fields -e frame.number -e ip.src -e http.cookie -e http.file_data
```

### Output (Excerpt)

```text
935 192.168.1.47 _ga=GA1.2.64336164626565663432.1700000000; _session=eyJhbGciOiJIUzI1NiJ9; _csrf=random_token_here 7469746c65...
945 192.168.1.51 _ga=GA1.2.e65be57e1e5f49cd8207.1700000000; _session=fake 7469746c65...
955 192.168.1.61 _ga=GA1.2.9749beffbfb832917307.1700000000; _session=fake 7469746c65...
965 192.168.1.52 _ga=GA1.2.c5ef4d29136f15c7782d.1700000000; _session=fake 7469746c65...
```

### Analysis

Frames 945, 955, and 965 are decoys from other internal IPs (192.168.1.51, .61, .52). 
Frame 935 is from Marcus (192.168.1.47). Looking closely at his _ga (Google Analytics) 
cookie, the client ID is a hex string: 64336164626565663432.

### Command 2: Extract ICMP packets to non-gateway IPs

```bash
$ tshark -r dd3_hard.pcap -Y "icmp && ip.addr == 192.168.1.47" -T fields -e frame.number -e ip.src -e ip.dst -e icmp.type -e icmp.seq -e data.data
```

### Output (Excerpt)

```text
1182 192.168.1.47 45.33.32.156 8 1 
1183 45.33.32.156 192.168.1.47 0 1 
1188 192.168.1.47 45.33.32.156 8 2 41d954fd080666665702120013103d06
1189 45.33.32.156 192.168.1.47 0 2 41d954fd080666665702120013103d06
1192 192.168.1.47 45.33.32.156 8 3 41d954fd080ccccd0d525a5c575f3e1c
1193 45.33.32.156 192.168.1.47 0 3 41d954fd080ccccd0d525a5c575f3e1c
1196 192.168.1.47 45.33.32.156 8 4 41d954fd081333330d173a0b00411000
1197 45.33.32.156 192.168.1.47 0 4 41d954fd081333330d173a0b00411000
1200 192.168.1.47 45.33.32.156 8 5 
1201 45.33.32.156 192.168.1.47 0 5 
```

### Analysis

Marcus is sending 5 Echo Requests (type 8) to 45.33.32.156. The data field 
contains hex payloads. Notice the repeating 41d954fd08 header, followed by the 
actual ciphertext. (Note: Wireshark's dissector missed the payload data on Seq 1 
and Seq 5, showing them as blank).

## PHASE 2: EXTRACTING THE DECRYPTION KEY

The _ga cookie contained a hex string. We decoded it using xxd.

### Command

```bash
$ echo "64336164626565663432" | xxd -r -p; echo
```

### Output

```text
d3adbeef42
```

### Methodology

"d3adbeef42" is our XOR decryption key. It's a play on "deadbeef" (a classic 
hex magic number) and "42" (the answer to everything). The key length is 10 bytes.

## PHASE 3: DECRYPTING THE ICMP PAYLOADS

We attempted to decrypt the hex payloads manually, but this failed due to 
Wireshark skipping frames 1182 and 1200. To get the absolute raw data, we wrote 
a Python script using the 'scapy' library.

The Algorithm:
1. Read the PCAP file.
2. Filter for IP packets from 192.168.1.47 to 45.33.32.156 with ICMP type 8.
3. Extract the raw payload (p[Raw].load).
4. Strip the first 8 bytes (the 41d954fd08 timestamp/header).
5. Sort the payloads by their ICMP sequence number (1 through 5).
6. Concatenate the payloads.
7. XOR the concatenated ciphertext byte-by-byte against the key "d3adbeef42".

### The Python Script
<img width="756" height="382" alt="image" src="https://github.com/user-attachments/assets/fff21c32-ba6b-4d3e-b79c-a520a4a7848c" />

```python
$ python3 - << 'EOF'
from scapy.all import rdpcap, IP, ICMP, Raw

# Read the pcap
pkts = rdpcap("dd3_hard.pcap")
chunks = []

for p in pkts:
    if p.haslayer(IP) and p.haslayer(ICMP) and p.haslayer(Raw):
        # Filter for Marcus's outbound Echo Requests to the specific IP
        if p[IP].src == "192.168.1.47" and p[IP].dst == "45.33.32.156" and p[ICMP].type == 8:
            # Strip the 8-byte ICMP timestamp header from each packet
            chunks.append((p[ICMP].seq, p[Raw].load[8:]))

# Sort by ICMP sequence number and concatenate
chunks.sort(key=lambda x: x[0])
ct = b"".join(data for _, data in chunks)

# XOR Decryption
key = b"d3adbeef42"
flag = bytes(b ^ key[i % len(key)] for i, b in enumerate(ct))

print("Full flag:", flag.decode(errors="ignore"))
EOF
```

### Output

```text
Full flag: OO{1cmp_c0v3rt_ch4nn3l_xor_m4st3r}beef42
```

## PHASE 4: CLEANING THE FLAG
The output showed the flag, but with a suffix: "beef42".

Why did this happen?
The final ICMP packet (Seq 5) contained 6 bytes of null-padding (\x00\x00\x00\x00\x00\x00). 
When these null bytes are XORed against the final 6 bytes of our key ("beef42"), 
the result is "beef42" in plaintext (since 0x00 XOR 0x62 = 0x62, etc.).

We discard the padding suffix, yielding the final clean flag:
OO{1cmp_c0v3rt_ch4nn3l_xor_m4st3r}

Result: CORRECT

## KEY TAKEAWAYS & LESSONS LEARNED

1. Split-Channel Exfiltration: Attackers often use a "decoy" channel to send 
   metadata (like keys) and a separate channel for the actual data. Always 
   correlate multiple protocols.
2. Cookie Analysis: Web cookies (like _ga, _session, _csrf) are frequent hiding 
   spots for hex-encoded strings. Always decode suspicious cookie values.
3. Tool Limitations: Wireshark/tshark's ICMP dissector can sometimes truncate or 
   hide payloads it deems non-standard. When extracting raw data, Scapy is often 
   more reliable because it reads the raw packet buffer directly.
4. XOR Padding: When XORing variable-length payloads, leftover null bytes in 
   the ciphertext will reveal the remaining key bytes in the plaintext output. 
   Always check the end of the decrypted string and trim the padding.
