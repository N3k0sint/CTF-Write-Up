# CTF Writeup: NO CALLER ID
<img width="618" height="518" alt="image" src="https://github.com/user-attachments/assets/8c4f9cdd-8f6c-4dcc-b63b-4b79a6a94bbb" />

| Field | Value |
|---|---|
| **Category** | `Audio Forensics / Signal Decoding` |
| **Difficulty** | `Easy` |
| **Points** | `100` |
| **Challenge File** | `No_Caller_ID.wav` |
| **Final Flag** | `OO{c4Ll_m3_mAyb3}` |

---

## Executive Summary

The challenge provides a WAV audio file containing a sequence of phone keypad tones. The hint suggests looking closer at the "sequence of sounds." This points to DTMF (Dual-Tone Multi-Frequency) decoding. Upon successfully decoding the DTMF tones into a string of digits, we discovered the digits mapped directly to ASCII decimal codes (e.g., 79 79 123...), which translated to the hidden flag.

## Phase 1: Initial Analysis and the Noisy Decoder

We started by attempting to decode the DTMF tones using a standard Python script utilizing the Fast Fourier Transform (FFT) to detect the low and high frequencies of the keypad.

### Initial Script (Naive Approach)

The script used a sliding 40ms window and an amplitude threshold (10% of max volume) to detect tone bursts.

```bash
python3 - << 'EOF'
# (Code used a simple sliding window and amplitude threshold)
EOF
```

### Output

```text
Decoded DTMF: 71917191213191915121716101819151019151915101916151219181512151
```

### Analysis of Failure

The output contained stray "1"s between every digit. This happened because the sliding window caught the quiet transitions between the actual tones. The background noise in those transition gaps caused the FFT to default to the first key ('1'). However, we could visually see the pattern "79 79 123" emerging.

## Phase 2: RMS Energy Detection (Second Attempt)

To fix the edge artifacts, we tried a different approach: calculating the rolling Root Mean Square (RMS) energy to find contiguous blocks of sound and analyzing only the middle 50% of each block.

### Second Script

```bash
python3 - << 'EOF'
# (Code used rolling RMS energy to find bursts, then analyzed the center)
EOF
```

### Output

```text
Clean DTMF: 5
Decoded Flag:
```

### Analysis of Failure

This approach failed completely because the background noise floor of the audio file was higher than the 15% amplitude threshold we set. The script merged all the tones into a single continuous block of sound, resulting in a single '5'.

## Phase 3: The Breakthrough (Spectral Ratio)

To successfully decode the file, we had to ignore the overall amplitude and instead look for the spectral ratio. DTMF tones have very specific frequencies (e.g., 697Hz + 1209Hz). We updated the script to use a Hanning window and check if the target frequencies stood out significantly above the median background noise floor (by a factor of 15x).

### Final Script

```python
import numpy as np
from scipy.io import wavfile

rate, data = wavfile.read("No_Caller_ID.wav")
if data.ndim > 1:
    data = data[:, 0]
data = data.astype(float)

LOW = [697, 770, 852, 941]
HIGH = [1209, 1336, 1477, 1633]
KEYS = [
    ['1', '2', '3', 'A'],
    ['4', '5', '6', 'B'],
    ['7', '8', '9', 'C'],
    ['*', '0', '#', 'D'],
]

win = int(rate * 0.04)   # 40ms window
step = int(rate * 0.01)  # 10ms step

raw_seq = []
for i in range(0, len(data) - win, step):
    seg = data[i:i+win] * np.hanning(win)
    freqs = np.fft.rfftfreq(len(seg), 1 / rate)
    mag = np.abs(np.fft.rfft(seg))
    noise = np.median(mag) + 1e-9

    low_mags = [mag[np.argmin(np.abs(freqs - f))] for f in LOW]
    high_mags = [mag[np.argmin(np.abs(freqs - f))] for f in HIGH]

    if max(low_mags) > noise * 15 and max(high_mags) > noise * 15:
        key = KEYS[np.argmax(low_mags)][np.argmax(high_mags)]
    else:
        key = '-'
    raw_seq.append(key)

# Collapse consecutive identical keys and ignore 1-frame glitches
tones = []
prev = '-'
count = 0
for k in raw_seq + ['-']:
    if k == prev:
        count += 1
    else:
        if prev != '-' and count >= 2:
            tones.append(prev)
        prev = k
        count = 1

dtmf = "".join(tones)
print("Decoded DTMF:", dtmf)

# Try ASCII decimal decoding
flag = ""
i = 0
while i < len(dtmf):
    if dtmf[i] == '1' and i + 3 <= len(dtmf) and 32 <= int(dtmf[i:i+3]) <= 126:
        flag += chr(int(dtmf[i:i+3]))
        i += 3
    elif i + 2 <= len(dtmf) and 32 <= int(dtmf[i:i+2]) <= 126:
        flag += chr(int(dtmf[i:i+2]))
        i += 2
    else:
        flag += f"[{dtmf[i]}]"
        i += 1

print("ASCII Decode:", flag)
```

### Output
<img width="743" height="583" alt="image" src="https://github.com/user-attachments/assets/b7a17578-6335-4979-8fde-e245d6feb4d6" />

```text
Decoded DTMF: 7979123995276108951095195109651219851125
ASCII Decode: OO{c4Ll_m3_mAyb3}
```

## Phase 4: Verification and Flag Submission

The decoded DTMF string was:

```text
7979123995276108951095195109651219851125
```

By parsing the string into valid ASCII decimal codes (2 or 3 digits), we get:

| ASCII Decimal | Character |
|---|---|
| 79 | O |
| 79 | O |
| 123 | { |
| 99 | c |
| 52 | 4 |
| 76 | L |
| 108 | l |
| 95 | _ |
| 109 | m |
| 51 | 3 |
| 95 | _ |
| 109 | m |
| 65 | A |
| 121 | y |
| 98 | b |
| 51 | 3 |
| 125 | } |

This translates directly to the flag.

**Flag Submitted:** `OO{c4Ll_m3_mAyb3}`  
**Result:** `CORRECT`

## Key Takeaways and Lessons Learned

1. **DTMF Decoding:** DTMF uses two simultaneous frequencies (one low, one high) to represent keys on a phone keypad. Tools like `multimon-ng` can automate this, but understanding the FFT math is crucial when writing custom scripts.
2. **Edge Artifacts:** When analyzing audio in chunks, transition periods between tones can pick up background noise or bleed from adjacent tones. Using a window function (like Hanning) and checking the spectral ratio against the noise floor is much more reliable than simple amplitude thresholds.
3. **ASCII Decimal:** Text can be encoded into audio by dialing the ASCII decimal values of each character. Always look out for numbers in the 32-126 range, which correspond to printable ASCII characters.
4. **Trial and Error:** The first two scripts failed due to background noise, but observing the pattern (e.g., 79 79 123) in the noisy output proved the methodology was correct and just needed refinement.
