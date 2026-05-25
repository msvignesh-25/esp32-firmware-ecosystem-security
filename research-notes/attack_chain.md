# Attack Chain — End to End Narrative
## ESP32 Firmware Security Analysis | Part 2

---

## Overview

This document narrates the complete attack chain executed during this project —
from initial physical access to full credential recovery, binary analysis, and
hardware security assessment. It is written from the attacker's perspective to
clearly communicate threat impact, and concludes with the defender's response.

This project is a direct continuation of Part 1, where the same ESP32 device
was used as a network attacker — running ARP poisoning, DNS spoofing, a
credential harvesting captive portal, and RF signal replay. Part 2 answers the
question: *what does an attacker learn if they capture that device?*

---

## Threat Model

**Attacker profile:**
- Basic Linux knowledge
- Free tools (esptool, binwalk, strings, Ghidra)
- Physical access to the device for 5-10 minutes
- No specialised hardware beyond a USB cable

**Target:**
- ESP32-WROOM 38-pin DevKit
- Running a credential harvesting sketch with hardcoded WiFi credentials
- No hardware security features enabled (factory default eFuse state)

**Attack goal:**
- Extract WiFi network credentials
- Understand the device's full attack capability
- Assess hardware security posture

---

## Stage 1 — Physical Access and Firmware Extraction

**Time required:** ~2 minutes  
**Tools:** esptool.py, USB cable  
**Authentication required:** None

The attacker connects the ESP32 to a laptop via USB. One command triggers the
ROM bootloader by pulling GPIO0 LOW and pulsing the EN reset pin. The bootloader
activates with no PIN, no password, no cryptographic challenge.

```
esptool.py --port /dev/ttyUSB0 --baud 115200 read_flash 0x0 ALL firmware.bin
```

4MB stream over UART. The complete flash contents — bootloader,
partition table, application firmware, file system — land in a single binary
file on the attacker's machine. The device continues operating normally.
The owner has no indication anything occurred.

**What the attacker now has:** An exact byte-for-byte copy of everything stored
on the device. Every secret the firmware holds is now on the attacker's laptop.

---

## Stage 2 — SPI Signal Capture (Attempted)

**Tools:** Saleae Logic Analyser, Logic 2  
**Result:** Partial — internal bus not accessible via header pins

A Saleae Logic Analyser was connected to the VSPI header pins (GPIO18, 19, 23, 5) with the intent of passively capturing SPI flash traffic during ESP32 boot.
The SPI analyzer was configured in Logic 2 with correct channel mapping and
active-low CS polarity.

On capture, signal transitions were observed on CLK, MOSI, and CS lines — but
no SPI protocol data was decoded. Investigation revealed that ESP32 boot-time
firmware reads occur over the internal SPI bus (GPIO6-11), which connects
directly between the ESP32 die and the onboard flash chip. These pins are not
broken out to the 38-pin header and require direct physical tapping.

**What is required for successful capture:**
- SOIC-8 test clip on the W25Q32/W25Q128 flash chip
- Or fine wire soldering directly to flash chip legs
- Saleae sample rate of at least 8 MHz to resolve SPI clock edges

**Why this matters despite the limitation:**
Even without the SPI capture, Stage 1 already achieved complete firmware
extraction via the software path. The SPI capture would have demonstrated
the same data at the hardware signal level — showing the firmware bytes
travelling over physical wires during boot. This is a hardware forensics
technique used when software extraction is blocked but physical access exists.

---

## Stage 3 — Binary Signature Analysis

**Time required:** ~30 seconds  
**Tools:** binwalk 2.4.3

The attacker runs binwalk against firmware.bin. In seconds, the complete flash
partition layout is recovered — no reverse engineering required, no guesswork.
The partition table is stored unencrypted at 0x8000 and binwalk reads it
directly.

Critical discoveries at this stage:

The application partition sits at 0x10000. HTML document signatures appear at
0x10314 — the captive portal page is embedded in plaintext. Unix filesystem
paths appear at 0x10CA0, revealing the development machine's OS, username, and
exact toolchain version. Three separate ESP32 image headers confirm the device
runs a dual-OTA firmware configuration.

**Attacker's position after Stage 3:** Complete map of the device's flash
layout. Exact offsets for every data region. Confirmed presence of embedded
HTML and developer artifacts before even opening the binary.

**Note on tooling:** binwalk 3.x (Go rewrite) was tested first and returned
only 5 results — missing the partition table, HTML headers, and path leaks
entirely. binwalk 2.4.3 returned 28 results. Tool version selection matters
significantly in binary analysis workflows.

---

## Stage 4 — Credential Recovery via String Analysis

**Time required:** ~10 seconds after firmware is in hand  
**Tools:** strings, grep, sed

This is the most impactful stage of the attack chain. The attacker runs:

```
strings firmware.bin | grep -i "Airtel"
```

The WiFi SSID appears immediately at line 181. Five lines later — the password,
sitting in plaintext between NVS field label markers. No decryption. No
specialised knowledge. A tool that ships with every Linux installation.

The same pass reveals the complete captive portal HTML source — POST endpoint,
form field names, Apple captive portal intercept path. An analyst examining
this device now knows its entire attack methodology without powering it on,
without connecting to its network, without any network traffic analysis.

The developer's machine path appears as a compiler artifact — full Linux path
including username, IDE version, and ESP32 core version. A hardcoded User-Agent
string identifying as Windows XP running Chrome 30 — a unique fingerprint
visible in any network traffic from this device.

**Total time from USB connection to credential recovery: under 3 minutes.**

This is the core finding of the project. Everything else — binwalk, Ghidra,
espefuse — provides depth, confirmation, and professional documentation. But
the attack itself is complete at this stage. Two commands. Three minutes.
Full network access.

---

## Stage 5 — Assembly Level Confirmation via Ghidra

**Tools:** Ghidra 11.x, Xtensa:LE:32:default processor module

The firmware binary was imported into Ghidra and analysed with the Xtensa
processor module — matching the ESP32's LX6 CPU architecture. Auto-analysis
completed in approximately 4 minutes.

Memory search for the SSID string located it at 5 separate addresses:
0x9944, 0x9d64, 0xb224, 0xb524, 0x10a92. All 5 references traced back to
a single compiled function — FUN_002004c8 — the stripped equivalent of the
Arduino setup() function.

In the disassembly panel, both SSID and password strings were directly visible
as string literals. The CPU loads raw pointers to these strings into Xtensa
registers before calling the WiFi connect function. No decryption call precedes
the load. No transformation is applied. The string goes from flash storage
directly to the WiFi stack unchanged.

This constitutes assembly-level proof of the credential storage vulnerability.
It rules out any possibility that the strings were obfuscated or transformed
at runtime — the disassembly shows they are used exactly as stored.

---

## Stage 6 — Hardware Security State Assessment

**Tools:** espefuse.py

espefuse.py read the complete eFuse configuration of the device. Every security
fuse returned factory default values:

- Flash encryption disabled — FLASH_CRYPT_CNT = 0
- Secure boot V1 disabled — ABS_DONE_0 = False
- Secure boot V2 disabled — ABS_DONE_1 = False
- UART download mode enabled — UART_DOWNLOAD_DIS = False
- JTAG debugging enabled — JTAG_DISABLE = False
- Encryption key slot empty — BLOCK1 = 00 × 32 bytes
- Secure boot key slot empty — BLOCK2 = 00 × 32 bytes

This is the factory default state. The device was deployed operationally —
performing network attacks in Part 1 — with every hardware security feature
disabled. This is not unusual. The vast majority of ESP32 devices shipped in
consumer and industrial products are in this exact state.

The implication is that every attack in this chain — firmware extraction,
credential recovery, binary analysis — works on essentially any unmodified
ESP32 in the field.

---

## Stage 7 — Vulnerability Summary

Four findings documented in CVE style:

| ID | Finding | Severity |
|----|---------|---------|
| VUL-01 | Hardcoded WiFi credentials in plaintext firmware | Critical |
| VUL-02 | Unauthenticated UART bootloader access | High |
| VUL-03 | Full application source reconstructible from binary | Medium |
| VUL-04 | No security eFuses burned — factory default state | High |

VUL-04 is the root cause. VUL-01, 02, and 03 are all consequences of a device
deployed without any hardware security configuration applied.

---

## Stage 8 — Defence and the Attacker's Dead End

**UART download mode restriction — demonstrated:**

```
esptool.py --port /dev/ttyUSB0 --baud 115200 --before no_reset read_flash 0x0 0x1000 test.bin
```

With the bootloader trigger skipped, the ESP32 remained in sketch execution
mode. When esptool sent its handshake, the ESP32 responded with sketch UART
output — byte 0x9C — instead of a bootloader acknowledgment. Fatal error.
No data extracted.

On a device with UART_DOWNLOAD_DIS eFuse burned permanently, this failure
cannot be circumvented by any software means. The ROM itself ignores the
bootloader trigger. Stage 1 of this entire attack chain becomes impossible.

Flash encryption adds a second layer — even if the flash chip were physically
desoldered and read with a dedicated flash programmer, the contents would be
AES-256 ciphertext. Strings analysis yields nothing. Ghidra shows encrypted
data. The attack chain collapses at Stage 1 with no path forward.

Secure boot closes the firmware replacement vector — a hardened device will
only execute firmware signed with the developer's private key. An attacker
who somehow bypasses extraction and attempts to flash modified firmware finds
the device refuses to boot it.

Three eFuse operations. All free. All supported in hardware on every ESP32.
None of them enabled by default. All of them required before deployment.

---

## Timeline Summary

| Time | Action | Result |
|------|--------|--------|
| T+0:00 | USB connected, esptool started | Bootloader triggered |
| T+2:00 | read_flash complete | 4MB firmware.bin on attacker laptop |
| T+2:30 | binwalk firmware.bin | Full partition map recovered |
| T+2:40 | strings + grep SSID | WiFi SSID found — line 181 |
| T+2:50 | strings + sed context | WiFi password recovered — plaintext |
| T+3:00 | strings HTML grep | Full captive portal source recovered |
| T+5:00 | Ghidra import + analysis | SSID at 5 memory addresses confirmed |
| T+6:00 | espefuse.py summary | All security fuses confirmed open |

**Total time to full credential recovery: under 3 minutes.**  
**Total time to complete security assessment: under 10 minutes.**  
**Specialised hardware required: None.**  
**Cost of attack tools: Zero.**

---

## Conclusion

This project demonstrates that an ESP32 device running operationally with
hardcoded credentials and no hardware security configuration is fully
compromised by any attacker with 5 minutes of physical access and a laptop.

The vulnerability is not in the ESP32 hardware — the chip provides robust
security features. The vulnerability is in deployment practice. Security
features that are opt-in and disabled by default will remain disabled unless
developers explicitly understand and apply them.

The tools used in this project — esptool, binwalk, strings, Ghidra — are
free, openly documented, and pre-installed or easily installed on any Linux
system. The barrier to performing this attack is a USB cable.

Hardware-level protections are the only reliable defence. They are available,
free to implement, and permanent once applied. The ethical obligation to apply
them before deploying any device that handles sensitive credentials is
unambiguous.

---

*Part of a multi-stage IoT security home lab series.*  
*← [Part 1: Multi-Layer IoT Network Attack Chain](https://github.com/msvignesh-25/layer2-to-physical-attack-chain)*  
*This document: Part 2 — Firmware Extraction and Security Analysis*