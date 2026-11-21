# CTF Writeup: DISGRUNTLEDDEV II

<img width="602" height="712" alt="image" src="https://github.com/user-attachments/assets/2a612150-2de7-4c3f-a373-b214f27f4acd" />

| Field | Value |
|---|---|
| **Category** | `Network Forensics / PCAP Analysis` |
| **Difficulty** | `Medium` |
| **Points** | `50` |
| **Author** | `digitalsurf` |
| **Target** | `Marcus (192.168.1.47)` |
| **PCAP File** | `dd2_medium.pcap` |
| **Final Flag** | `OO{dns_tunn3l_l1k3_4_pr0}` |

---

## EXECUTIVE SUMMARY

The challenge involves a disgruntled developer, Marcus, who is exfiltrating data 
from his workstation. IT has blocked standard FTP/file transfers, but the SOC 
detected unusual DNS traffic. By analyzing the provided PCAP file (dd2_medium.pcap), 
we discovered that Marcus used DNS Tunneling to smuggle out Base32-encoded data 
via TXT records to an external domain (totallylegit.io). After extracting and 
reassembling the chunks, we decoded the Base32 string to reveal the flag.

## PHASE 1: INITIAL TRAFFIC ANALYSIS

We started by opening the PCAP file and filtering for traffic originating from 
Marcus's IP address (192.168.1.47) to isolate his actions from background noise.

### Command Run

```bash
$ tshark -r dd2_medium.pcap -Y "ip.src == 192.168.1.47 && dns.qry.name contains \"totallylegit.io\"" -T fields -e dns.qry.name | sort -u
```

### Output

```text
heartbeat.totallylegit.io
J5HXWZDOONPXI5LONYZW.0.data.cdn-analytics.totallylegit.io
status.totallylegit.io
YX3MGFVTGXZUL5YHEMD5.1.data.cdn-analytics.totallylegit.io
```

### Analysis of Output

Most traffic was standard background noise (Google, GitHub, internal Jira). 
However, the queries to "totallylegit.io" stood out. 
- "heartbeat" and "status" look like C2 (Command and Control) check-ins.
- The two queries with ".data.cdn-analytics." prefixes contain long, random 
  alphanumeric strings, followed by sequence numbers (.0. and .1.).
- The character set of the strings (A-Z, 2-7) immediately suggests Base32 encoding.

## PHASE 2: DATA EXTRACTION AND REASSEMBLY

The exfiltrated data was split across two DNS queries. We extracted the chunks 
based on their sequence numbers (.0 and .1) and concatenated them.

### Chunk 0 (Sequence 0)

```text
J5HXWZDOONPXI5LONYZW
```

### Chunk 1 (Sequence 1)

```text
YX3MGFVTGXZUL5YHEMD5
```

### Concatenated Payload

```text
J5HXWZDOONPXI5LONYZWYX3MGFVTGXZUL5YHEMD5
```

## PHASE 3: THE DECODING METHODOLOGY (AND THE MANUAL PITFALL)

Algorithm & Initial Failure:
Base32 uses a specific alphabet (A-Z and 2-7). Every 8 characters (40 bits) 
decode to 5 bytes of plaintext data. 

Method:
To ensure absolute accuracy, we piped the exact concatenated string into the Linux 
base32 utility. The utility accurately processes the bitstream without manual 
boundary errors.

<img width="577" height="72" alt="image" src="https://github.com/user-attachments/assets/5323a187-bef2-4da4-961b-1fbaf31484cf" />


### Command Run

```bash
$ echo "J5HXWZDOONPXI5LONYZWYX3MGFVTGXZUL5YHEMD5" | base32 -d; echo
```

### Output

```text
OO{dns_tunn3l_l1k3_4_pr0}
```

## PHASE 4: VERIFICATION AND SUBMISSION

We verified the output format matched the required CTF flag format (OO{...}).
Flag Submitted: OO{dns_tunn3l_l1k3_4_pr0}
Result: CORRECT

## KEY TAKEAWAYS & LESSONS LEARNED

1. DNS Tunneling Recon: When analyzing PCAPs, always look for TXT, NULL, or 
   CNAME records pointing to external domains. High-entropy subdomains (random 
   strings) are a dead giveaway for data exfiltration.
2. Sequence Numbers: Always check for numerical prefixes/suffixes in subdomains 
   (like .0. and .1.) to properly order and reassemble the data.
3. Base32 Recognition: If a string only contains uppercase letters A-Z and 
   numbers 2-7, it is almost certainly Base32.
4. Trust Automation Over Manual Decoding: Manually decoding Base32 bit-by-bit 
   is extremely error-prone. Always use standard CLI tools (like base32 -d) or 
   Python scripts to extract the exact plaintext.
