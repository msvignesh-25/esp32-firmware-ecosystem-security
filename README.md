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
- 
