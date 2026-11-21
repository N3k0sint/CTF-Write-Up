# CTF Writeup: HIDDEN IN THE STREAMS 2
<img width="597" height="700" alt="image" src="https://github.com/user-attachments/assets/5a2818a1-7f8c-4421-b989-6480ec2b73e2" />

| Field | Value |
|---|---|
| **Category** | `Forensics / Disk Image Analysis` |
| **Difficulty** | `Hard` |
| **Points** | `75` |
| **Author** | `quut` |
| **Challenge File** | `backup.dd.gz` |
| **Final Flag** | `OO{H1Dd3n_!n_7h3_57r3@m5!1}` |

---

## EXECUTIVE SUMMARY

The challenge provides a compressed disk image (backup.dd.gz) and a hint: 
"NTFS files can hold more data than what you can see..." This points directly to 
NTFS Alternate Data Streams (ADS). By decompressing the image and using The 
Sleuth Kit's 'fls' tool, we discovered hidden data streams attached to both an 
active file (flag.wav) and a deleted file (flag.txt). Extracting these streams 
revealed Base64-encoded chunks that, when concatenated and decoded, provided the 
final flag.

## PHASE 1: DECOMPRESSION AND IMAGE ANALYSIS

First, we decompressed the provided backup and identified the file type to 
determine if we needed to calculate partition offsets.

### Commands Run

```bash
$ ls
backup.dd.gz dd2_medium.pcap dd3_hard.pcap dd4_insane.pcap

$ gunzip -k backup.dd.gz
$ file backup.dd
$ mmls backup.dd
```

### Output

```text
backup.dd: DOS/MBR boot sector, code offset 0x52+2, OEM-ID "NTFS ", sectors/cluster 8, Media descriptor 0xf8, sectors/track 0, dos < 4.0 BootSector (0x80), FAT (1Y bit by descriptor); NTFS, sectors 40959, $MFT start cluster 4, $MFTMirr start cluster 2559, bytes/RecordSegment 2^(-1*246), clusters/index block 1, serial number 017014c0366a6cf1e
```

### Analysis

The 'file' command identifies the raw image as an NTFS file system directly, 
not a full disk with a partition table (mmls output is omitted/empty). This 
means we can interact with the image directly without specifying a sector offset.

## PHASE 2: ENUMERATING FILES AND ALTERNATE DATA STREAMS

To find the hidden data, we used 'fls' (from The Sleuth Kit) to recursively list 
all files, deleted files (marked with *), and their associated attribute streams. 
In NTFS, Alternate Data Streams appear as filename:streamname.

### Command Run

```bash
$ fls -r -p backup.dd | grep ":"
```

### Output

```text
r/r 4-128-1: $AttrDef
r/r 8-128-2: $BadClus
r/r 8-128-1: $BadClus:$Bad
r/r 6-128-1: $Bitmap
r/r 7-128-1: $Boot
d/d 11-144-2: $Extend
r/r 25-144-2: $Extend/$ObjId:$O
r/r 24-144-3: $Extend/$Quota:$O
r/r 24-144-2: $Extend/$Quota:$Q
r/r 26-144-2: $Extend/$Reparse:$R
r/r 2-128-1: $LogFile
r/r 0-128-1: $MFT
r/r 1-128-1: $MFTMirr
r/r 9-128-2: $Secure:$SDS
r/r 9-144-3: $Secure:$SDH
r/r 9-144-4: $Secure:$SII
r/r 10-128-1: $UpCase
r/r 10-128-2: $UpCase:$Info
r/r 3-128-3: $Volume
r/r 65-128-2: flag.wav
r/r 65-128-4: flag.wav:part0
r/r 65-128-5: flag.wav:part1
r/r 65-128-6: flag.wav:part2
r/r 65-128-7: flag.wav:part3
r/r 65-128-8: flag.wav:part4
-/r * 64-128-2: flag.txt
-/r * 64-128-10: flag.txt:0
-/r * 64-128-11: flag.txt:1
-/r * 64-128-12: flag.txt:2
-/r * 64-128-13: flag.txt:3
-/r * 64-128-14: flag.txt:4
-/r * 72-128-2: normalfile.txt
V/V 73: $OrphanFiles
-/r * 16: $OrphanFiles/OrphanFile-16
-/r * 17: $OrphanFiles/OrphanFile-17
-/r * 18: $OrphanFiles/OrphanFile-18
-/r * 19: $OrphanFiles/OrphanFile-19
-/r * 20: $OrphanFiles/OrphanFile-20
-/r * 21: $OrphanFiles/OrphanFile-21
-/r * 22: $OrphanFiles/OrphanFile-22
-/r * 23: $OrphanFiles/OrphanFile-23
```

### Analysis

This reveals the smoking gun. There are two interesting files:
1. flag.wav (Inode 65) - Active file with streams named part0 through part4.
2. flag.txt (Inode 64) - Deleted file (marked with *) with streams named 0 through 4.
We also see normalfile.txt (Inode 72).

## PHASE 3: EXTRACTING THE HIDDEN STREAMS

We used 'icat' to extract the data from the specific Inode-Attribute IDs.
(Note: An attempted mount with ntfs-3g failed due to missing kernel support for 
the 'streams_interface' parameter, but 'icat' works perfectly without mounting).

### Command Run

```bash
$ icat backup.dd 64-128-2; echo
$ for i in 10 11 12 13 14; do icat backup.dd 64-128-$i; echo; done

$ for i in 4 5 6 7 8; do icat backup.dd 65-128-$i > wav_part_$i; done
$ file wav_part_*
$ head -c 200 wav_part_*; echo

$ icat backup.dd 72-128-2; echo
```

### Output

```text
=== flag.txt (main) ===
Tm90aGluZyB0byBzZWUgaGVyZS4=

=== flag.txt stream 64-128-10 ===
T097SDFE
=== flag.txt stream 64-128-11 ===
ZDNuXyFu
=== flag.txt stream 64-128-12 ===
XzdoM181
=== flag.txt stream 64-128-13 ===
N3IzQG01
=== flag.txt stream 64-128-14 ===
ITF9

wav_part_4: ASCII text, with no line terminators
wav_part_5: ASCII text, with no line terminators
wav_part_6: ASCII text, with no line terminators
wav_part_7: ASCII text, with no line terminators
wav_part_8: ASCII text, with no line terminators

==> wav_part_4 <==
T097SDFE
==> wav_part_5 <==
ZDNuXyFu
==> wav_part_6 <==
XzdoM181
==> wav_part_7 <==
N3IzQG01
==> wav_part_8 <==
ITF9

=== normalfile.txt ===
Nothing to see here. 
```

### Analysis

- The main body of the deleted flag.txt decodes to "Nothing to see here." (Tm90aGluZyB0byBzZWUgaGVyZS4=)
- The streams for both flag.txt and flag.wav contain identical Base64 strings.
- The strings end with '=' and '}', which are classic Base64 padding and ending characters.

## PHASE 4: DECODING THE FLAG

We concatenated the five Base64 chunks from the extracted streams:
T097SDFE + ZDNuXyFu + XzdoM181 + N3IzQG01 + ITF9
= T097SDFEZDNuXyFuXzdoM181N3IzQG01ITF9

We then piped this string into the 'base64' decoder:

### Command Run
<img width="557" height="635" alt="image" src="https://github.com/user-attachments/assets/786fc209-a03d-4a53-8157-dcecd807b2e9" />

```bash
$ echo "T097SDFEZDNuXyFuXzdoM181N3IzQG01ITF9" | base64 -d; echo
```

### Output

```text
OO{H1Dd3n_!n_7h3_57r3@m5!1}

Flag Submitted: OO{H1Dd3n_!n_7h3_57r3@m5!1}
Result: CORRECT
```

## KEY TAKEAWAYS & LESSONS LEARNED

1. NTFS Alternate Data Streams (ADS): NTFS allows files to have multiple data 
   streams. Standard directory listings (like ls or dir) do not show them. This 
   is a common technique for hiding malware or, in this case, CTF flags.
2. Deleted Files Still Hold Data: The 'fls' command showed that the deleted 
   flag.txt still had its streams intact. When files are deleted in NTFS, the 
   MFT entries are marked as unallocated, but the raw data often remains on disk 
   until overwritten.
3. The Power of 'icat': You do not need to mount a disk image to extract files. 
   The Sleuth Kit's 'icat' command allows direct extraction of specific streams 
   using their Inode-Attribute ID (e.g., 64-128-10).
4. Recognize Base64: The payload strings (T097SDFE, ZDNuXyFu, etc.) contain 
   typical Base64 characters (A-Z, a-z, 0-9, +, /) and the final chunk ended in 
   '}', indicating the end of a flag.
