# HackerVerse CTF - September 2026 - Level 1: Cryptic Canvas

| Field | Details |
|---|---|
| **Category** | Digital Forensics & Steganography |
| **Platform** | HackerVerse (EC-Council) |
| **Total Challenges** | 5 |
| **Completed** | 5/5 |

### 1. The Hidden Voice

**Scenario:**
An audio file named Audio.wav was recovered during an investigation. It is 
suspected that the file contains a hidden message encoded using Morse code.
Artifact Location: /root/Stego Artifacts/Audio.wav

**Forensics Analysis:**
1. Inspecting the file shows it is an 8-bit mono PCM audio file running at 
   11,050 Hz with a carrier tone at 1,000 Hz.
2. The audio is not direct Morse-to-text; it encodes binary digits:
   - "0" is transmitted as Morse digit "0" (-----)
   - "1" is transmitted as Morse digit "1" (.----)
3. Using multimon-ng with a 1,000 Hz bandpass filter extracts the clean 
   bitstream directly without timing errors.
4. Parsing the bitstream at an offset of 6 bits reveals the ASCII flag string.

**Commands & Solution:**

```bash
cd "/root/Stego Artifacts"
```

```bash
sudo apt update && sudo amultimon-ng soxpt install -y
```

```bash
sox Audio.wav -esigned-integer -b16 -r 22050 -t raw - bandpass 1000 100 | multimon-ng -t raw -a MORSE_CW -
```

**Decoded Bits:**
1 0 0 1 1 0 0 1 1 0 1 1 0 0 0 1 1 0 0 0 0 1 0 1 1 0 0 1 1 1 0 1 1 1 1 0 1 1 0 1 1 0 0 0 1 0 0 1 1 0 1 0 0 1 0 1 1 0 1 1 1 0 0 1 1 0 0 0 0 1 0 1 1 1 0 0 1 0 0 1 1 1 1 0 0 1 0 1 0 1 1 1 1 1 0 1 1 0 0 0 0 1 0 1 1 1 0 1 0 1 0 1 1 0 0 1 0 0 0 1 1 0 1 0 0 1 0 1 1 0 1 1 1 1 0 1 0 1 1 1 1 1 0 1 1 1 0 0 1 1 0 1 1 0 0 1 0 1 0 1 1 0 0 0 1 1 0 1 1 1 0 0 1 0 0 1 1 0 0 1 0 1 0 1 1 1 0 1 0 0 0 1 1 1 0 0 1 1 0 1 1 1 1 1 0 1

**Run Python to decode to ASCII:**

```bash
python3 -c '
bits = "10011001101100011000010110011101111011011000100110100101101110011000010111001001111001010111110110000101110101011001000110100101101111010111110111001101100101011000110111001001100101011101000111001101111101"
print("".join(chr(int(bits[6:][i:i+8], 2)) for i in range(0, len(bits[6:])-7, 8)))
'
```

**FLAG:** flag{binary_audio_secrets}

### 2. JPEG SOI Marker

**Scenario:**
What is the hexadecimal marker that starts a JPEG File Interchange Format file?
Format: XX XN XX XN

**Solution:**
A standard JPEG JFIF file begins with the Start of Image (SOI) marker (FF D8) 
followed by the APP0 JFIF application marker (FF E0).

**ANSWER:** FF D8 FF E0

### 3. Stego Cat

**Scenario:**
Recover the flag hidden within a corrupted JPEG image named Cat.jpg and decode it.
Artifact Location: /root/Stego Artifacts/Cat.jpg

**Forensics Analysis:**
1. Running "file Cat.jpg" returns "data" because the header magic bytes are broken.
2. Inspecting the first 16 bytes reveals "00 d8 ff e0" instead of "ff d8 ff e0".
3. Repairing the first 4 bytes restores the JFIF format.
4. Running steghide extraction with an empty passphrase extracts flag.txt.
5. Base64-decoding flag.txt gives the flag.

**Commands & Solution:**

```bash
cd "/root/Stego Artifacts"
```

```bash
xxd -p -l 16 Cat.jpg
```

**Output:** `00d8ffe000104a464946...`

**Repair header:**

```bash
python3 -c '
with open("Cat.jpg", "r+b") as f:
    f.seek(0)
    f.write(bytes.fromhex("ffd8ffe0"))
'
```

**Extract payload using steghide:**

```bash
steghide extract -sf Cat.jpg -p ""
```

```bash
cat flag.txt
```

**Output:** `Q1RGe3N0ZWcwXzFzX2NvMGx9`

**Decode Base64 string:**

```bash
echo "Q1RGe3N0ZWcwXzFzX2NvMGx9" | base64 -d
```

**FLAG:** CTF{steg0_1s_co0l}

### 4. Camouflage - Sequence Identification

**Scenario:**
Select the decoding sequence revealed by the second hint string inside 
Encoded_Dump.txt.
Artifact Location: /root/Encoded Artifact/Encoded_Dump.txt

**Forensics Analysis:**
1. Encoded_Dump.txt contains Base64-encoded strings between transmission tags.
2. Line 5 reveals a reversed decoy: ".eno tsuj si eurt ,syoced era owT"
   (Reversed: "Two are decoys, true is just one.")
3. Line 6 reveals the recipe hint:
   "Process: Base58 → ROT13 → BASE64 → XOR (Hex Key: 342120)"

**ANSWER:** Base58 → ROT13 → Base64 → XOR (Hex Key: 342120)

### 5. Camouflage - Final Secret Flag

**Scenario:**
What is the final hidden flag after decoding the paste dump in Encoded_Dump.txt?
Format: FLAG{XxXxXxXxXxXxXxX}

**Forensics Analysis:**
Passing candidate Line 3 through the exact sequence (Base58 -> ROT13 -> Base64 
-> XOR with hex key 342120) decrypts the secret flag.

**Commands & Solution:**

```bash
cd "/root/Encoded Artifact"
```

```bash
python3 - << 'EOF'
import base64
import codecs

B58_CHARS = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"
def b58_dec(s):
    n = 0
    for c in s:
        n = n * 58 + B58_CHARS.index(c)
    h = hex(n)[2:]
    if len(h) % 2: h = '0' + h
    res = bytes.fromhex(h)
    pad = len(s) - len(s.lstrip('1'))
    return b'\x00' * pad + res

def b64_clean(s):
    if isinstance(s, bytes):
        s = s.decode('latin1')
    s = s.strip()
    return base64.b64decode(s + '=' * ((4 - len(s) % 4) % 4))

key = bytes.fromhex("342120")

with open("Encoded_Dump.txt") as f:
    lines = [l.strip() for l in f if l.strip() and not l.startswith("---")]

for idx, line in enumerate(lines, 1):
    # Try the candidate string both directly and after an initial b64 peel
    candidates = [line]
    try:
        candidates.append(b64_clean(line).decode('latin1'))
    except:
        pass

    for cand in candidates:
        try:
            # 1. Base58
            p1 = b58_dec(cand.strip()).decode('latin1')
            # 2. ROT13
            p2 = codecs.decode(p1, 'rot_13')
            # 3. Base64
            p3 = b64_clean(p2)
            # 4. XOR (Hex Key: 342120)
            flag = bytes([b ^ key[i % len(key)] for i, b in enumerate(p3)]).decode('utf-8', errors='ignore')

            if "FLAG{" in flag or "flag{" in flag:
                print(f"\n[+] SUCCESS on Line {idx}!")
                print(f"[+] Flag: {flag}\n")
        except Exception:
            pass
EOF
```

**FLAG:** FLAG{ViCtOrYaChIeVeD}

## Final Level 1 Summary

**Challenge 1:** `flag{binary_audio_secrets}`

**Challenge 2:** `FF D8 FF E0`

**Challenge 3:** `CTF{steg0_1s_co0l}`

**Challenge 4:** `Base58 → ROT13 → Base64 → XOR (Hex Key: 342120)`

**Challenge 5:** `FLAG{ViCtOrYaChIeVeD}`
