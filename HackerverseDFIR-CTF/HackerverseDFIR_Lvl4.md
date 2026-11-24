# HackerVerse CTF - Level 4: Ransomware Analysis and Decryption

## Scenario

<img width="766" height="482" alt="image" src="https://github.com/user-attachments/assets/89b840be-197e-495b-bbd1-5aaf8a72b7cf" />

A user on the Windows Victim VM reported that files in
C:\Users\Admin\Documents\CTF_Vault were encrypted by ransomware. A ransom
window demanded payment. Our task: analyze the sample, determine the
encryption key, and decrypt Flag.txt.locked.

| Field | Details |
|---|---|
| **Victim VM** | Windows (user: `Admin`) |
| **Artifact** | `Rasmine.exe` (masquerading as a Sysinternals tool) |

---

### 1. Full Path of Ransomware Executable

**QUESTION:**
  What is the full path provided for the Rasome executable?

**METHOD:**
  Search the drive for the ransomware binary.

**Command:**
    dir /s /b C:\*Rasom*.exe

**Output:**
    C:\Users\Admin\AppData\Local\Rasome\Rasome.exe
    C:\Users\Admin\Desktop\sysinternals\Rasom.exe
    C:\Users\Admin\Desktop\sysinternals\Rasome.exe

  The full path with persistence location (AppData\Local) is the intended
  answer.

**ANSWER:**
  ```C:\Users\Admin\AppData\Local\Rasome\Rasome.exe```

---

### 2. .NET Framework Version

**QUESTION:**
  What version of the .NET Framework is linked to the packed executable?

**METHOD A - PowerShell assembly metadata:**
  [System.Reflection.Assembly]::LoadFile(
      "C:\Users\Admin\Desktop\sysinternals\Rasom.exe"
  ).ImageRuntimeVersion

**Output:** v4.0.30319

**METHOD B - Detect It Easy:**

<img width="505" height="340" alt="image" src="https://github.com/user-attachments/assets/fed46b08-b8af-448a-99d7-4d17b2d0feb2" />


  Drag Rasom.exe into DIE.
  Library line shows: .NET(v4.0.30319)

**ANSWER:**
  ```v4.0.30319```

---

### 3. Packer Identification

**QUESTION:**
  What packer was used to compress/obfuscate the Rasome executable?

**METHOD - Detect It Easy:**
  1. Launch "Detect It Easy.lnk" from Desktop
  2. Drag Rasom.exe into the window
  3. Read the Packer field
     
<img width="505" height="340" alt="image" src="https://github.com/user-attachments/assets/fed46b08-b8af-448a-99d7-4d17b2d0feb2" />

**Output:**
    Packer: MPRESS(1.27)[-]
    Library: .NET(v4.0.30319)

**Corroborating evidence - Windows Prefetch showed:**
    C:\Windows\Prefetch\RASOME.EXE-5A5248CF.pf
    C:\Windows\Prefetch\RASOME.EXE-ED3CEB46.pf
    C:\Windows\Prefetch\RASOME_UNPACKED.EXE-AC674ED1.pf
  (the UNPACKED prefetch proves the packer self-extracts at runtime)

**ANSWER:**
```MPRESS```


---

### 4. Key Generation Function

**QUESTION:**
  What is the name of the function used to generate the encryption key?

**METHOD:**
  MPRESS hides the .NET IL in a compressed section. The unpacked assembly
  exists only in memory. Use dnSpy live attach to extract it.

  Step 1: Launch the ransomware
    C:\Users\Admin\Desktop\sysinternals\Rasome.exe

  Step 2: In dnSpy, attach to the running process
    Debug -> Attach to Process -> Rasome.exe -> Attach

  Step 3: Open the Modules window (Ctrl+Alt+U)
    Locate the "Rasom.exe" module with NO path (memory-only = unpacked).
    Right-click -> Save Module -> save as Rasom_unpacked.exe

  Step 4: Detach from the process, then:
    File -> Open -> Rasom_unpacked.exe

  Step 5: Search for key generation functions
    Ctrl+Shift+K -> search "Generate"

**Function found in the cryptographic module:**
```csharp
public static string GenerateKeyFromMachineNameMD5(string machineName)
{
    using (MD5 md = MD5.Create())
    {
        byte[] bytes = Encoding.UTF8.GetBytes(machineName);
        byte[] hash  = md.ComputeHash(bytes);
        return BitConverter.ToString(hash)
                           .Replace("-", "")
                           .ToUpperInvariant();
    }
}
```

**ANSWER:**
  ```GenerateKeyFromMachineNameMD5```

---

### 5. Calculated Decryption Key

**QUESTION:**
  What is the calculated decryption key for the analyzed machine?

**ALGORITHM ANALYSIS:**

**The function GenerateKeyFromMachineNameMD5:**
    1. Takes the Windows machine name (ComputerName / HostName)
    2. Computes MD5 of its UTF-8 bytes
    3. Returns the 32-character UPPERCASE hex digest

  The ransom note confirmed the cipher: AES-128
    "All your documents ... have been encrypted with strong AES-128 encryption."

  AES-128 requires a 16-byte key. The MD5 digest (16 raw bytes) IS that key.

**STEP 1 - Confirm the machine name:**

**Command:**
    hostname
    echo %COMPUTERNAME%

**Output:**

WIN10

WIN10

  (Also confirmed by LSASS dump: "domainname WIN10")

STEP 2 - Compute MD5("WIN10"):

**PowerShell:**
```powershell
$md5 = [System.Security.Cryptography.MD5]::Create()
$hash = $md5.ComputeHash(
    [System.Text.Encoding]::UTF8.GetBytes("WIN10")
)
([BitConverter]::ToString($hash) -replace '-','').ToUpper()
```

**OR Python:**
```bash
python -c "import hashlib; print(hashlib.md5(b'WIN10').hexdigest().upper())"
```

**Output:**
    037E39A5C57380EA9357167684CA4DD7

**VERIFICATION:**
  Paste the key into the ransomware GUI ("Enter the decryption key below")
  and click DECRYPT FILES. The ransomware responds:
    "Decryption process completed. Check the target directory for your files."

  This confirms the key is correct.

**ANSWER:**
  ```037E39A5C57380EA9357167684CA4DD7```

---

### 6. Decrypting Flag.txt.locked

**QUESTION:**
  What is the final flag obtained after decrypting Flag.txt?

METHOD A - Use the ransomware's own decryptor (recommended):

<img width="743" height="592" alt="image" src="https://github.com/user-attachments/assets/73ce7cb5-a9f6-4e77-9dfe-fe9d13926609" />

  1. The ransom window is still open (or relaunch Rasome.exe)
  2. Paste the key into the input field:
       037E39A5C57380EA9357167684CA4DD7
  3. Click "DECRYPT FILES"
  4. A dialog confirms: "Decryption process completed."
  5. The file Flag.txt.locked is decrypted in place to Flag.txt
  6. Read the flag:
       type "C:\Users\Admin\Documents\CTF_Vault\Flag.txt"

     <img width="637" height="375" alt="image" src="https://github.com/user-attachments/assets/df84a5bb-cb09-4ec8-b3ce-2191b69ee4e6" />


**METHOD B - Manual decryption with Python:**

  The malware uses AES-128. The key is the raw 16-byte MD5 digest
  (NOT the hex string -- use .digest(), not .hexdigest()).

  Script (decrypt.py):
```python
from Cryptodome.Cipher import AES
from Cryptodome.Util.Padding import unpad
import hashlib

key = hashlib.md5(b"WIN10").digest()   # 16 raw bytes

data = open(
    r"C:\Users\Admin\Documents\CTF_Vault\Flag.txt.locked", "rb"
).read()

for name, mode, iv in [
    ("ECB", AES.MODE_ECB, None),
    ("CBC", AES.MODE_CBC, b"\x00"*16),
]:
    cipher = AES.new(key, mode) if iv is None else AES.new(key, mode, iv=iv)
    pt = unpad(cipher.decrypt(data), 16)
    print(f"[{name}] {pt.decode('utf-8', errors='replace')}")
```

**Output:**
    [ECB] flag{b4CKup_SaV3D_7h3_$YsTeM}

**ANSWER:**
  ```flag{b4CKup_SaV3D_7h3_$YsTeM}```

---

## Attack Chain Summary

  1. Locate sample ................ `C:\Users\Admin\Desktop\sysinternals\Rasom.exe`
  2. Identify packer (DIE) ........ `MPRESS 1.27`
  3. Check .NET runtime ........... `v4.0.30319`
  4. Unpack via dnSpy live attach . `save Rasom_unpacked.exe`
  5. Decompile with dnSpy ......... `find GenerateKeyFromMachineNameMD5`
  6. Identify machine name ........ `WIN10 (hostname / LSASS dump)`
  7. Compute key .................. `MD5("WIN10")`
                                    = `037E39A5C57380EA9357167684CA4DD7`
  8. Feed key into GUI ............ `files decrypted automatically`
  9. Read decrypted Flag.txt ...... `flag{b4CKup_SaV3D_7h3_$YsTeM}`

## Key Takeaways

  1. Packers obfuscate, they don't protect. MPRESS-packed .NET binaries
     still load the original IL into memory at runtime. Live debugging
     (dnSpy attach) beats static analysis for packed .NET malware.

  2. Always read the ransom note. The text said "AES-128", narrowing the
     key search to 16 bytes and ruling out AES-256 candidates.

  3. Weak key derivation breaks ransomware. Using MD5(machineName) as an
     AES key is a fatal design flaw -- machine names are public info.

  4. Compare ransom IDs across runs. The ID stayed identical, proving the
     key is deterministic (machine-derived), not random.

  5. The malware's own decryptor is a shortcut. Once the key is known,
     feeding it to the GUI is faster than reimplementing the algorithm.

  6. Toolchain:
       Detect It Easy  -> packer identification
       dnSpy           -> .NET decompiler + live debugger
       PowerShell      -> assembly metadata (ImageRuntimeVersion)
       procdump        -> full memory dump (fallback)
       DotDumper       -> automated in-memory .NET assembly extraction

## Final Answer Sheet

**Challenge 1:** `C:\Users\Admin\AppData\Local\Rasome\Rasome.exe`

**Challenge 2:** `v4.0.30319`

**Challenge 3:** `MPRESS`

**Challenge 4:** `GenerateKeyFromMachineNameMD5`

**Challenge 5:** `037E39A5C57380EA9357167684CA4DD7`

**Challenge 6:** `flag{b4CKup_SaV3D_7h3_$YsTeM}`

## End Of Level 4 Writeup

Thankyou for Reading! :D
