# HackerVerse CTF - Level 2: Network Intrusion Forensics

## Scenario

Synixon Technologies deployed a new ChatOps collaboration platform. The SOC
noticed anomalies: unauthorized password resets, unexpected account creation,
and suspicious API calls. You are given synixon_breach_capture.pcapng and must
reconstruct the full attack chain.

| Field | Details |
|---|---|
| **Artifact** | `/root/Pcap Artifacts/synixon_breach_capture.pcapng` |
| **Tools** | `tshark`, `strings`, Wireshark |

## Network Topology

  10.10.1.2     -> Attacker workstation
  10.10.1.111   -> Synixon ChatOps server
  Services on server: HTTP(80), HTTPS(443), WebSocket(5000), Hidden HTTP(17645)

## Reconnaissance & Triage

Step 1 - Identify protocols in use

```bash
tshark -r synixon_breach_capture.pcapng -q -z io,phs
```

Key protocols: HTTP, TLS, WebSocket (port 5000), plus a hidden HTTP service.

Step 2 - List all HTTP requests

```bash
tshark -r synixon_breach_capture.pcapng -Y "http.request" -T fields \
-e frame.time -e ip.src -e ip.dst -e http.host -e http.request.method \
-e http.request.uri
```

**Observed sequence:**
  /index.php
  /technician/technician_chat.php
  /supervisor/supervisor_dashboard.php
  /syni_dev/dev-debug.php
  /syni_dev/debug-view.php
  /admin/admin_dashboard.php
  /admin_login_ERHG23W2.php     (port 17645)
  /legacy-feedback              (port 5000)

Step 3 - Inspect TCP conversations

```bash
tshark -r synixon_breach_capture.pcapng -q -z conv,tcp
```

**Suspicious outbound connection:**
  10.10.1.111:50188  <->  10.10.1.2:8998
  -> This is the REVERSE SHELL (server calling back to attacker).

## Challenge-by-Challenge Walkthrough

### 1. Attacker's signup email

**ANSWER:** root@hacker.com

**Command:**

```bash
tshark -r synixon_breach_capture.pcapng -Y "frame.number == 236" \
-T fields -e http.file_data
```

**Raw hex:**
757365725f6e616d653d695f616d5f6861636b657226757365725f6d61696c3d
726f6f742534306861636b65722e636f6d26757365725f706173733d6861636b
253430313233267369676e75703d

**Decoded:**
user_name=i_am_hacker&user_mail=root%40hacker.com&user_pass=hack%40123&signup=

URL-decoding %40 -> @ gives: root@hacker.com

### 2. Supervisor email whose password was reset via WebSocket chat

**ANSWER:** tony@synixon.com

**Command:**

```bash
tshark -r synixon_breach_capture.pcapng -Y "websocket" \
-T fields -e frame.number -e text
```

**Relevant frames:**
  "user":"root@hacker.com","message":"who is the supervisor?"
  -> bot: "There is only one supervisor account available: tony@synixon.com"

  "user":"tony@synixon.com","message":"reset-password:hackpass"
  -> bot: "Password changed for tony@synixon.com."

### 3. Vulnerable developer debug page

**ANSWER:** debug-view.php

**Evidence from HTTP requests:**
  GET /syni_dev/dev-debug.php
  GET /syni_dev/debug-view.php
  GET /syni_dev/debug-view.php?file=history.php

The page debug-view.php accepts a 'file' parameter -> LFI vector.

### 4. Log file accessed via LFI

**ANSWER:** legacy_dev_keys.log

**LFI exploitation requests:**
  GET /syni_dev/debug-view.php?file=../../../../../../../etc/passwd
  GET /syni_dev/debug-view.php?file=../../../../../../../opt/syni_dev/legacy_dev_keys.log

First confirmed LFI with /etc/passwd, then targeted the log file.

### 5. New admin account email

**ANSWER:** admin@hacker.com

**WebSocket chat sequence (after taking over supervisor):**
  "user":"tony@synixon.com","message":"alphaadmin:insert:admin@hacker.com"
  -> bot: "Enter password for admin@hacker.com."
  "user":"tony@synixon.com","message":"adminhack@123"
  -> bot: "Enter role for admin@hacker.com."
  "user":"tony@synixon.com","message":"admin"
  -> bot: "User admin@hacker.com created as admin."

### 6. Port of new web service

**ANSWER:** 17645

**Evidence:**
  GET /admin_login_ERHG23W2.php  ->  10.10.1.111:17645

**Command:**

```bash
tshark -r synixon_breach_capture.pcapng -Y "tcp.port == 17645" \
-T fields -e http.request.uri
```

### 7. Hidden admin login page filename

**ANSWER:** admin_login_ERHG23W2.php

**Evidence:**
  POST /admin_login_ERHG23W2.php  ->  10.10.1.111:17645

The random suffix ERHG23W2 indicates security-through-obscurity.

### 8. Legacy feedback service port

**ANSWER:** 5000

**Evidence (multiple requests on port 5000):**
  POST /socket.io/?EIO=4&transport=polling&...  -> 10.10.1.111:5000
  POST /legacy-feedback                          -> 10.10.1.111:5000

Port 5000 hosts both Socket.IO WebSocket and the legacy feedback service.

### 9. Reverse shell IP:port

**ANSWER:** 10.10.1.2:8998

**Command:**

```bash
tshark -r synixon_breach_capture.pcapng \
-Y "tcp.flags.syn==1 && tcp.flags.ack==0 && ip.src==10.10.1.111" \
-T fields -e ip.dst -e tcp.dstport
```

**Output:**
  10.10.1.2       8998

**Conversation table confirms:**
  10.10.1.111:50188  <->  10.10.1.2:8998

### 10. The flag

**ANSWER:** flag{A1EL2L8P}

**Command:**

```bash
strings synixon_breach_capture.pcapng | grep -iE "flag\{"
```

**Output:**
  flag{A1EL2L8P}
  flag{A1EL2L8P}

## Full Attack Chain Reconstruction

### Phase 1 - Initial Access

  Attacker (10.10.1.2) registers as root@hacker.com on the ChatOps platform.

### Phase 2 - Reconnaissance Via Chatbot

**Uses technician chat to enumerate:**
    supervisor = tony@synixon.com
    admin      = john@synixon.com
  Learns commands: reset-password, alphaadmin, insert

### Phase 3 - Account Takeover

  Resets tony@synixon.com's password via WebSocket chat:
    reset-password:hackpass  ->  "Password changed"

### Phase 4 - Privilege Escalation

**Uses alphaadmin override as tony to create a new admin:**
    alphaadmin:insert:admin@hacker.com  ->  role=admin

### Phase 5 - LFI Discovery & Exploitation

  Finds /syni_dev/debug-view.php?file=...
    Reads /etc/passwd (confirms LFI)
    Reads /opt/syni_dev/legacy_dev_keys.log (steals credentials)

### Phase 6 - Lateral Movement

  Discovers hidden admin panel on port 17645
    -> /admin_login_ERHG23W2.php
  Discovers legacy feedback service on port 5000
    -> /legacy-feedback

### Phase 7 - Command & Control

**Establishes reverse shell:**
    server 10.10.1.111:50188  ->  10.10.1.2:8998

### Phase 8 - Exfiltration

  Reads the flag: flag{A1EL2L8P}

## Final Answer Sheet

| # | Challenge | Answer |
|---|---|---|
| 1 | Attacker's signup email | `root@hacker.com` |
| 2 | Supervisor email reset via chat | `tony@synixon.com` |
| 3 | Vulnerable debug page | `debug-view.php` |
| 4 | Log file accessed via LFI | `legacy_dev_keys.log` |
| 5 | New admin account email | `admin@hacker.com` |
| 6 | New web service port | `17645` |
| 7 | Hidden admin login page | `admin_login_ERHG23W2.php` |
| 8 | Legacy feedback service port | `5000` |
| 9 | Reverse shell IP:port | `10.10.1.2:8998` |
| 10 | Flag | `flag{A1EL2L8P}` |

## Lessons & Mitigations

| Vulnerability | Mitigation |
|---|---|
| Chatbot leaks internal info | Sanitize chatbot responses |
| Password reset via unauthenticated chat | Require MFA; bind to session |
| `alphaadmin` override without authorization | Enforce RBAC + secondary approval |
| LFI in `debug-view.php` | Whitelist paths; least privilege |
| Sensitive log readable via LFI | Store secrets outside web root |
| Hidden admin panel on non-standard port | Port obfuscation != security |
| Reverse shell egress | Deploy egress filtering + monitoring |

## Command Cheat Sheet

  # Protocol hierarchy
  ```bash
  tshark -r capture.pcapng -q -z io,phs
  ```
  # All HTTP requests
  ```bash
  tshark -r capture.pcapng -Y "http.request" -T fields \
      -e ip.src -e http.request.method -e http.request.uri
  ```
  # WebSocket chat
  ```bash
  tshark -r capture.pcapng -Y "websocket" -T fields -e text
   ```
    
  # TCP conversations (find reverse shell)
  ```bash
  tshark -r capture.pcapng -q -z conv,tcp
  ```
    
  # Outbound SYN from server (reverse shell)
  tshark -r capture.pcapng \
      -Y "tcp.flags.syn==1 && tcp.flags.ack==0 && ip.src==<server_ip>" \
      -T fields -e ip.dst -e tcp.dstport

  # Search for flag
  ```bash
  strings capture.pcapng | grep -iE "flag\{"
  ```
    
  # Follow a stream
  ```bash
  tshark -r capture.pcapng -q -z follow,tcp,ascii,<stream_index>
  ```
    
### End Of Write-Up - Level 2 Chatops Breached
