# ESP32 Firmware Security Analysis
### IoT Attack Surface Project - Part 2 | Firmware Extraction, Reverse Engineering & Hardening 

> A comprehensive, firmware-level security analysis of an ESP32 device. This project covers end-to-end firmware extraction, physical SPI flash bus auditing, binary reverse engineering and hardware-level mitigation simulation.

---

## ⚠️ Disclaimer
This project was built and tested exclusively on personally owned devices in an isolated home lab environment. All techniques demonstrated are for educational and security awareness purposes only. Performing these attacks on networks or devices without explicit permission is illegal.

---

## Project Overview
* **Part 1** of this series demonstrated what the ESP32 can execute as an offensive network adversary-carrying out multi-layer attacks involving ARP poisoning, DNS spoofing, captive portal credential harvesting and sub-GHz frame manipulation.
* **Part 2** shifts focus to a critical defensive, threat-modeling question: *What happens if an attacker gets their hands on the physical device?*

Using standard, open-source utilities and direct physical USB/UART interface access, this project extracts the complete 4MB flash layout, recovers the hardcoded Wi-Fi credentials in plaintext, reconstructs the application architecture via disassembly inside Ghidra, maps unauthenticated bootloader channels and outlines hardware-enforced silicon defenses.

---

## Attack & Defense Chain

```text
Physical USB Access
        │
        ▼
┌─────────────────────┐
│  Stage 1            │  esptool.py read_flash ──> 4MB raw firmware.bin extracted
│  Firmware Extraction│
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Stage 2            │  Passive SPI bus capture during boot (Saleae Logic Analyzer)
│  SPI Signal Capture │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Stage 3            │  binwalk signature analysis ──> partition layout & assets mapped
│  Binary Analysis    │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Stage 4            │  strings parsing ──> Plaintext SSIDs, passwords, & source leaks
│  String Analysis    │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Stage 5            │  Ghidra (Xtensa LE) ──> Memory mapping & setup() disassembly
│  Reverse Engineering│
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Stage 6            │  espefuse.py ──> Silicon register audit (All security eFuses open)
│  UART Bootloader    │
└────────┬────────────┘
         │
         ▼
┌──────────────────────┐
│  Stage 7             │  CVE-style vulnerability report ──> 4 distinct findings mapped
│  Vulnerability Report│
└────────┬─────────────┘
         │
         ▼
┌─────────────────────┐
│  Stage 8            │  Non-destructive validation of Secure Boot & Flash Encryption
│  Defense Simulation │
└─────────────────────┘

```
---

## Hardware Used

|       Component      |                                   Purpose                                       |
|----------------------|---------------------------------------------------------------------------------|
| Kali Linux           | Host OS                                                                         |
| ESP32                | Target Node                                                                     |
| Salae Logic Analyzer | Utilized for passive over-the-air and line-level SPI sniffing during boot phase |

---

## Software/ Tools Used

- Extraction: `esptool.py v5.2.0` (Flash dump and bootloader communications)
- Silicon Audit: `espfuse.py v5.2.0` (eFuse state verification)
- Static Analysis: `binwalk v2.4.3` (Signature analysis and binary parsing)
- String Recovery: `GNU binutils` (*strings* & *grep*)
- SRE Framework: `Ghidra v11.x` (Xtensa 32-bit architecture disassembly and decompilation support)

---

## Project Structure

```text
|
├── README.md
|
├── 

```

---

## Proof of Concept

### Stage 1 ─ Firmware Extraction
- Executed a raw 4MB flash memory read over the serial bus. The target device required zero physical or cryptographic authentication challenges prior to dumping storage blocks.<br>Done through the command `esptool.py --port /dev/ttyUSB0 --baud 115200 read_flash 0x0 ALL firmware.bin`

### Stage 2 ─ Passive SPI Capture
- Connected a Logic Analyzer directly to the onboard SPI flash interface lines. Captured raw transmission transitions at boot time, proving that binary instructions transit the physical circuit board tracks completely unencrypted.

### Stage 3 ─ Binary Structure Analysis
- Used `binwalk` to scan the raw binary file. Reconstructed the flash partition table layout and identified invididual application image headers.

### Stage 4 ─ Credentials Revealed
- Passed the raw binary data segments using GNU utilities like *strings* & *grep*. A single extended regular expression could extract the plaintext WiFi credentials and also the full HTML source code of the captive portal.<br>Command used: `strings firmware.bin | grep -E -i "ssid|password"`

### Stage 5 ─ Ghidra Reverse Engineering
- Imported the application parition block into Ghidra using the Xtensa LE 32-bit processor definitions. Traced cross-references for the identified credential string offsets, pointing directly back to the data definitions inside the initialization blocks of the compiled `setup()` routine.<br>Used the GNU utility `sed` to filter the text

### Stage 6 ─ UART Bootloader Access Validation
- Ran an inspection of the internal eFuse mapping blocks (Refer to the image). The silicon registers returned and it's confirmed that zero cryptographic hardware protections were initialized.

### Stage 7 ─ Cyber-Physical Vulnerability Assessment

* Compiled four distinct security findings modeled closely after standard enterprise CVE metrics:

|                 Title                      | Severity|            CVSS v3.1 Vector                 |        Impact Summary                 |
|--------------------------------------------|---------|---------------------------------------------|---------------------------------------|
| Plaintext WiFi credentials in Static Memory| Critical| CVSS:3.1/AV:P/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:N| Exposure of organizational network entry points |
| Unauthenticated UART Bootloader Interface  | High    | CV33:3.1/AV:P/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H| Device firmware integiry compromised  |
| Factory-Default Open Silicon eFuse State   | High    | CV33:3.1/AV:P/AC:H/PR:N/UI:N/S:U/C:H/I:H/A:H| Lack of hardware-enforced roots of trust |
| Cleartext Web Infrastructure Assembly      | Medium  | CV33:3.1/AV:P/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N| Full reconstruction of backend application endpoints|

- AV: Attack Vector; AC: Attack Complexity; PR: Privileges Required; UI: User Interaction; S:Scope; C:Confidentiality; I:Integrity; A:Availability
- The metric *Scope* has its value as **Changed**, which means that the attacker can also compromise the entire WiFi network due to credentials cpature.

### Stage 8 ─ Defensive Mitigations & Firmware Lockdown

To avoid destructive, permanent burning of the Dev Kit's eFuses, hardening behaviors were verified through software-level environement simulation.

- **Before Hardening:** Open-source utilities can communicate with the chip's boot ROM interface, allowing anyone to freely read from or write to any part of the flash memory without any restriction.
- **Simulated Hardening State:** Forcing the bootloader to restriction flags (`--before no_reset`), simulates how a secure device would behave when its ROM interface disables automatic communication. The terminal shows a fatal connection error, blocking any access to device's data.
- Documented the execution flows required to permanently burn the eFuses for production deployments

---

## Key Findings
1. In-transital confidential data can be extracted from non-encrypted binaries with Open-Source tools.
2. This entire extraction and mapping workflow can be deployed on a standard Raspberry Pi Zero W, making it highly effective for on-site physical breaches.

---

## Skills Demonstrated
- Embedded Microcontroller Firmware Extraction & Reconstruction
- Binary RE & Disassembly (`Ghidra`/Xtensa assembly interpretation)
- Serial Bus (SPI/UART) interception
- Cryptographic Hardening Analysis

---

**This project belongs to a series focused on IoT security frameworks.<br>« [Part 1 ─ Multi-Layer Cyber-Physical Network Attack Chain](https://github.com/msvignesh-25/layer2-to-physical-attack-chain)**