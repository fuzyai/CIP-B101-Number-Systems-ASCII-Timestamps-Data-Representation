# CIP-B101 Lab 1a – Digital Forensics Fundamentals: Number Systems, Hex/ASCII Encoding and Timestamp Conversion

**Student:** Fuseini Imoru Kantuogaa  
**Course:** CIP-B101  
**Lab:** Lab 1a – Binary/Hex/Decimal Conversion, Hexdump Analysis, Text-to-Hex Encoding and Unix/Mac Epoch Timestamp Arithmetic

## Overview

This lab demonstrates the foundational numeric and encoding skills that underpin all later forensic work in this course: converting between binary, hexadecimal and decimal using shell arithmetic, `bc`, and Python; inspecting and reconstructing raw bytes with `xxd`; encoding and decoding ASCII text to and from hexadecimal both at the shell level and programmatically in Python; generating a personal evidence file and verifying it byte-for-byte with hex dumps; converting Unix epoch timestamps to human-readable dates (and back) in both bash and Python, including UTC vs. local time zone handling; converting between the Mac/Cocoa epoch (2001-01-01) and the Unix epoch (1970-01-01); and confirming line-ending differences and system byte order, which matter when comparing file hashes or interpreting multi-byte values in forensic evidence. The goal was to build fluency with the low-level representations (bits, bytes, hex, epoch seconds) that every later forensic tool in this course operates on under the hood.

## Case Folder Structure

```
mkdir -p ~/cip-b101-lab1a
cd ~/cip-b101-lab1a
pwd
```

| Output | Value |
|---|---|
| Working directory | /home/kali/cip-b101-lab1a |

Unlike the later network/disk-forensics labs, this lab is a self-contained exercises lab with no acquired evidence image; all "evidence" files referenced below were generated locally as part of the exercises themselves.

## 1. Binary and Hexadecimal to Decimal Conversion (Bash Arithmetic)

```
echo $((2#101010))
echo $((16#AB2))
```

| Expression | Base | Result |
|---|---|---|
| `2#101010` | Binary | 42 |
| `16#AB2` | Hexadecimal | 2738 |

Bash's `$((base#value))` arithmetic-expansion syntax directly evaluates a number in any base from 2–64 without needing an external tool, confirming 101010₂ = 42₁₀ and AB2₁₆ = 2738₁₀.

## 2. Decimal to Binary Conversion (bc)

```
echo "obase=2;126" | bc
```

| Expression | Result |
|---|---|
| 126 in binary | 1111110 |

`bc`'s `obase` variable sets the output base for subsequent calculations — here converting 126₁₀ to 1111110₂. An initial attempt to use `printf "%X` (missing the closing quote and argument) produced an incomplete command, visible as the shell dropping into a `dquote>` continuation prompt waiting for the unterminated string to close; this was abandoned in favor of the Python approach below, which handles all four conversions cleanly in one block.

## 3. Cross-Verification with Python

```python
print(int('101010', 2))
print(int('AB2', 16))
print(bin(126))
print(hex(2738))
```

```
python3 - <<'PY'
...
PY
```

| Call | Output |
|---|---|
| `int('101010', 2)` | 42 |
| `int('AB2', 16)` | 2738 |
| `bin(126)` | 0b1111110 |
| `hex(2738)` | 0xab2 |

Every result here independently confirms the bash/bc conversions in sections 1–2, giving three separate tools (bash arithmetic, `bc`, Python) that all agree — a basic form of the cross-tool corroboration used throughout later labs.

## 4. Text-to-Hex Encoding with xxd

```
echo -n "Hello" > evidence_text.txt
cat evidence_text.txt
xxd evidence_text.txt
xxd -p evidence_text.txt
```

| Command | Output |
|---|---|
| `cat` | Hello |
| `xxd` (annotated hex dump) | `00000000: 4865 6c6c 6f  Hello` |
| `xxd -p` (plain hex) | 48656c6c6f |

The `-n` flag to `echo` suppressed the trailing newline, so the file contains exactly 5 bytes (H-e-l-l-o, hex `48 65 6c 6c 6f`), matching the ASCII values for each letter.

```
printf "Hello"
printf "48656c6c6f" | xxd -r -p
```

Both commands correctly reproduced `Hello` — the first by printing the literal string directly, and the second by reversing the hex-to-binary conversion (`xxd -r -p`, "reverse, plain hex") back into the original ASCII text, confirming the encode/decode round-trip is lossless.

## 5. Text-to-Hex Encoding and Decoding in Python (binascii)

```python
import binascii
text = b'Hello'
print('ASCII bytes:', text)
print('ASCII to hex:', binascii.hexlify(text).decode())
print('Hex to ASCII:', binascii.unhexlify('48656c6c6f').decode())
print('ord(H):', ord('H'))
print('chr(72):', chr(72))
```

| Call | Output |
|---|---|
| ASCII bytes | b'Hello' |
| ASCII to hex | 48656c6c6f |
| Hex to ASCII | Hello |
| ord('H') | 72 |
| chr(72) | H |

This confirms the same hex/ASCII mapping produced by `xxd` above, this time using Python's `binascii` module, and additionally confirms the single-character code point for 'H' (72 decimal = 0x48), matching the first byte seen in every hex dump in this lab.

## 6. Generating and Verifying a Personal Evidence File

```
echo "Digital Forensics Begins With Bytes." > Fuseini_lab1a.txt
cat Fuseini_lab1a.txt
xxd Fuseini_lab1a.txt
xxd -p Fuseini_lab1a.txt
xxd -r -p <(xxd -p Fuseini_lab1a.txt)
```

| Output | Value |
|---|---|
| File content | Digital Forensics Begins With Bytes. |
| `xxd` dump (first bytes) | `00000000: 4469 6774 6174 6c20 466f 7265 6e73 6963  Digital Forensic` |
| `xxd -p` (plain hex) | 4469676974616c20466f72656e736963732042656769 6e7320576974682042797465732e0a |
| Round-trip (`xxd -r -p` of its own `xxd -p` output, via process substitution) | Digital Forensics Begins With Bytes. |

This exercise generated a personal, named evidence file, dumped it to both annotated and plain hexadecimal, and then fed the plain hex dump straight back into `xxd -r -p` using bash process substitution (`<(...)`) to reconstruct the original text in a single pipeline, with no intermediate file needed — confirming the file's byte content round-trips exactly, including its trailing newline (hex `0a` at the end of the dump).

## 7. Unix Epoch Timestamp Conversion

```
date +%s
date -u -d @1643643491
```

| Command | Output |
|---|---|
| `date +%s` (current epoch at time of lab) | 1782253076 |
| `date -u -d @1643643491` | Mon Jan 31 03:38:11 PM UTC 2022 |

```python
from datetime import datetime, timezone
print(datetime.fromtimestamp(1643643491, tz=timezone.utc))
```

| Output | 2022-01-31 15:38:11+00:00 |
|---|---|

The bash `date -u -d @<epoch>` command and Python's `datetime.fromtimestamp(..., tz=timezone.utc)` independently agree on the exact same UTC timestamp for epoch value 1643643491, confirming both tools correctly interpret Unix epoch seconds (seconds since 1970-01-01 00:00:00 UTC).

## 8. Local Time vs. UTC

```
date
date -u
```

```python
from datetime import datetime, timezone
print('Local:', datetime.now())
print('UTC  :', datetime.now(timezone.utc))
```

| Source | Output |
|---|---|
| `date` (local) | Tue Jun 23 06:19:24 PM EDT 2026 |
| `date -u` (UTC) | Tue Jun 23 10:19:24 PM UTC 2026 |
| Python `datetime.now()` (local, no tz) | 2026-06-23 18:19:24.956634 |
| Python `datetime.now(timezone.utc)` | 2026-06-23 22:19:24.956854+00:00 |

The local-to-UTC offset is consistently 4 hours (18:19 EDT = 22:19 UTC), matching EDT's known UTC-4 offset, and both bash and Python agree on the same wall-clock values to within a sub-second margin (expected, since each tool call queries the system clock independently a fraction of a second apart). This confirms the lab machine's local time zone is set to U.S. Eastern Daylight Time, which is an important detail for later labs where timestamp evidence must be interpreted correctly against UTC.

## 9. Mac (Cocoa) Epoch vs. Unix Epoch Conversion

```python
from datetime import datetime, timezone
unix_epoch = datetime(1970, 1, 1, tzinfo=timezone.utc)
mac_epoch = datetime(2001, 1, 1, tzinfo=timezone.utc)
print(int((mac_epoch - unix_epoch).total_seconds()))
mac_value = 725846400
unix_value = mac_value + 978307200
print(datetime.fromtimestamp(unix_value, tz=timezone.utc))
```

| Step | Result |
|---|---|
| Offset between Mac epoch (2001-01-01) and Unix epoch (1970-01-01) | 978,307,200 seconds |
| Sample Mac-epoch value | 725,846,400 |
| Converted to Unix epoch | 725,846,400 + 978,307,200 = 1,704,153,600 |
| Converted to date (UTC) | 2024-01-02 00:00:00+00:00 |

This confirms the standard conversion constant between Apple's Core Foundation / Cocoa epoch (seconds since 2001-01-01 00:00:00 UTC, used in macOS/iOS artifacts such as Safari history and Property List timestamps) and the Unix epoch used everywhere else in this course: **Unix seconds = Mac/Cocoa seconds + 978,307,200**. This conversion is essential whenever examining macOS or iOS forensic artifacts whose timestamps are natively stored in Cocoa epoch format rather than Unix epoch.

## 10. Line-Ending Differences and System Byte Order

```
echo "Hello" > hello_with_newline.txt
echo -n "Hello" > hello_without_newline.txt
xxd hello_with_newline.txt
xxd hello_without_newline.txt
```

| File | Hex dump | Length |
|---|---|---|
| hello_with_newline.txt | `00000000: 4865 6c6c 6f0a  Hello.` | 6 bytes (trailing `0a` = LF) |
| hello_without_newline.txt | `00000000: 4865 6c6c 6f  Hello` | 5 bytes |

```
md5sum hello_with_newline.txt hello_without_newline.txt
sha256sum hello_with_newline.txt hello_without_newline.txt
```

| File | MD5 | SHA-256 |
|---|---|---|
| hello_with_newline.txt | 09f7e02f1290be211da707a266f153b3 | 66a045b452102c59d840ec097d59d9467e13a3f34f6494e539ffd32c1bb35f18 |
| hello_without_newline.txt | 8b1a9953c4611296a827abf8c47804d7 | 185f8db32271fe25f561a6fc938b2e264306ec304eda518007d1764826381969 |

This is a critical forensic lesson: two files containing what looks like "the same text" (Hello) produce **completely different hash values** because of a single invisible trailing newline byte. Any forensic hash comparison (for example, verifying a suspect file against a known hash) must account for exact byte content, not just visually apparent text — a one-byte difference invisible to the naked eye in a text editor completely changes both the MD5 and SHA-256 digests.

```python
import sys
print('System byte order:', sys.byteorder)
```

| Output | System byte order: little |

This confirms the lab machine's processor architecture uses little-endian byte ordering, meaning the least-significant byte of a multi-byte value is stored first in memory. This matters directly for forensic analysis of binary structures (such as registry hive timestamps, PE file headers, or raw multi-byte integers carved from memory, as seen in earlier labs) — misreading a value's byte order would silently produce an incorrect number, so confirming the host's native endianness is a necessary baseline before interpreting any raw multi-byte binary evidence.

## Conclusion

**This lab established the core numeric and encoding literacy required for all later forensic work in this course: reliable, cross-verified conversion between binary, hexadecimal and decimal; lossless round-trip encoding/decoding of text to and from hex at both the shell and Python level; and correct handling of the two timestamp formats (Unix epoch and Mac/Cocoa epoch) and the two time references (local and UTC) that recur throughout digital forensic artifacts.** The single most practically important finding was in section 10: two files differing only by an invisible trailing newline byte produce entirely different cryptographic hashes, which is the exact mechanism by which forensic tools detect even the smallest alteration to evidence — and a reminder that hash verification must always be performed on exact byte content, never on visual inspection of a file's displayed text.

## Key Tools Used

| Tool | Purpose |
|---|---|
| bash `$((base#value))` arithmetic | Direct binary/hex-to-decimal conversion |
| bc (`obase=`) | Decimal-to-arbitrary-base conversion |
| python3 (`int()`, `bin()`, `hex()`) | Cross-verification of all base conversions |
| xxd / xxd -p / xxd -r -p | Hex dump generation and reverse (hex-to-binary) reconstruction |
| printf | Literal string printing and alternate hex-to-text decoding |
| python3 binascii (`hexlify`/`unhexlify`) | Programmatic hex/ASCII encode-decode, `ord`/`chr` code-point lookup |
| date / date -u | Local and UTC timestamp display and epoch conversion |
| python3 datetime (`fromtimestamp`, `now`, `timezone.utc`) | Cross-verification of epoch/date conversions and local/UTC comparison |
| md5sum / sha256sum | Demonstrating hash sensitivity to exact byte content |
| python3 sys.byteorder | Confirming host endianness |

## Forensic Notes

- An early attempt at `printf "%X` without a closing quote or argument left the shell waiting in a `dquote>` continuation prompt; this is a normal interactive-shell artifact of an incomplete command (an unterminated double quote), not a tool malfunction, and was resolved by abandoning that incomplete line and proceeding with the Python-based conversion block instead.
- All four independent conversion methods used across sections 1–3 (bash arithmetic, `bc`, Python `int()`/`bin()`/`hex()`) produced identical results for the same values, which is the same cross-tool corroboration principle applied to disk/partition layout verification in other labs in this course.
- The Mac-epoch-to-Unix-epoch offset (978,307,200 seconds) is a fixed constant and should be memorized or scripted for quick reference whenever macOS/iOS artifacts (Safari history, Spotlight metadata, many Apple plist timestamps) are encountered in later casework.
- Section 10's hash comparison is the clearest, simplest demonstration in this lab of why forensic evidence handling always operates on exact bytes: a trailing newline that is completely invisible when the file is viewed with `cat` is fully visible in the `xxd` hex dump and produces a completely unrelated hash digest.
- This lab produced no case evidence folder structure (evidence/working/reports/etc.) because it is a self-contained fundamentals exercise with locally generated files rather than an acquired image or capture; the generated files (`evidence_text.txt`, `Fuseini_lab1a.txt`, `hello_with_newline.txt`, `hello_without_newline.txt`) remain in `~/cip-b101-lab1a` for reference.
- System byte order was confirmed little-endian on this host; this should be reconfirmed on any different host or architecture before interpreting raw multi-byte forensic artifacts, since big-endian systems (some embedded devices, certain network protocol fields) would require the opposite interpretation.
