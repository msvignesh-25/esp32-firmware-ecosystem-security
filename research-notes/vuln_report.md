# Vulnerability Report — ESP32 Firmware Security Analysis
**Project:** IoT Attack Surface Part 2 — Firmware Extraction & Reverse Engineering  
**Device:** ESP32-WROOM-32 38-pin DevKit

---

## VUL-01 — Hardcoded WiFi Credentials in Plaintext Firmware

**Severity:** Critical  
**CVSS v3.1 Vector:** AV:P/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N  
**CVSS Score:** 6.1 (Physical attack vector — score reflects physical access requirement)

### Description
WiFi network SSID and password are stored as plain text in the compiled application firmware. As no encryption is implemented, anyone with the physical access to the device can extract those credentials using standard Linux *command-line tools*.

### Location
- Memory address: `0x00009944` (SSID — primary occurrence)
- strings output: line 181 (SSID), line 184 (password)
- Adjacent to NVS field labels `sta.ssid` and `sta.pswd`

### Proof of Concept
```bash
# Step 1 — Extract firmware (requires physical USB access)
esptool.py --port /dev/ttyUSB0 --baud 115200 read_flash 0x0 ALL firmware.bin

# Step 2 — Recover SSID
strings firmware.bin | grep -i "Airtel"

# Step 3 — Recover password (context around SSID line)
strings firmware.bin | sed -n '175,195p'
```

**Result:** Full WiFi SSID and password printed to terminal in plaintext.

### Impact
- Full access to the target WiFi network
- Lateral movement to all devices on the same network
- Credential reuse attacks if the same password is used on other services
- Persistent network access — credentials remain valid until manually changed

### Remediation
- Never hardcode credentials in firmware source code
- Use ESP-IDF NVS encryption for credential storage
- Implement a provisioning workflow — credentials set post-deployment, never compiled in
- Enable flash encryption to prevent plaintext extraction even if device is captured

---

## VUL-02 — Unauthenticated UART Bootloader Access

**Severity:** High  
**CVSS v3.1 Vector:** AV:P/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H  
**CVSS Score:** 6.8

### Description
The main issue is that the ESP32's ROM bootloader accepts flash read, write and erase commands over UART completely unauthenticated. If someone has physical access to the device and trigger the standard bootloader sequence — dropping GPIO0 low while cycling the EN pin), they can gain full, unrestricted control over the entire flash memory.

### Location
- ROM bootloader — hardware level, not patchable via firmware update
- Accessible via `/dev/ttyUSB0` on Linux, `COMx` on Windows

### Proof of Concept
```bash
# Trigger bootloader and read hardware identification — no credentials required
esptool.py --port /dev/ttyUSB0 --baud 115200 flash_id

# Read full firmware — no credentials required
esptool.py --port /dev/ttyUSB0 --baud 115200 read_flash 0x0 ALL firmware.bin
```

**Result:** Hardware info returned immediately. Full 4MB firmware extracted in ~2 minutes.

### Impact
- Complete firmware extraction (leads to VUL-01, VUL-03, VUL-04)
- Firmware replacement — attacker can flash malicious firmware
- Flash erasure — evidence destruction
- Specific partition dump (NVS, SPIFFS, coredump)

### Remediation
- Burn `UART_DOWNLOAD_DIS` eFuse before deployment (irreversible — do it at your own risk)
- Command: `espefuse.py --port /dev/ttyUSB0 burn_efuse UART_DOWNLOAD_DIS`
- Note: Valid on ESP32 v3 and newer only

---

## VUL-03 — No Security eFuses Burned — Factory Default State

**Severity:** High  
**CVSS v3.1 Vector:** AV:P/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H  
**CVSS Score:** 6.8

### Description
All ESP32 security eFuses remain in factory default state. No hardware-level security features have been enabled. The device was deployed in an operational role (network attacker — see Part 1) without any embedded security configuration applied.

### eFuse State (confirmed via espefuse.py summary)

| eFuse | Value | Required State |
|-------|-------|---------------|
| FLASH_CRYPT_CNT | 0 | Odd bits set (encryption active) |
| ABS_DONE_0 | False | True (secure boot V1) |
| ABS_DONE_1 | False | True (secure boot V2) |
| UART_DOWNLOAD_DIS | False | True |
| JTAG_DISABLE | False | True |
| BLOCK1 (encryption key) | 00×32 | AES-256 key |
| BLOCK2 (secure boot key) | 00×32 | RSA/ECDSA public key digest |

### Impact
All attack vectors documented in VUL-01 through VUL-03 are exploitable as a direct consequences of VUL-04, which is the root cause — the absence of hardware security configuration enables all firmware-level attacks.

### Remediation
Refer to [hardening_guide](../defence/hardening_guide.md) for complete step-by-step hardening procedure including key generation, eFuse burn commands, and irreversibility warnings.

Priority order:
1. `UART_DOWNLOAD_DIS` — closes the extraction attack vector entirely
2. Flash encryption — renders extracted firmware unreadable
3. Secure boot — prevents firmware replacement attacks
4. `JTAG_DISABLE` — closes hardware debug attack vector

---

## VUL-04 — Full Application Source Reconstructible from Binary

**Severity:** Medium  
**CVSS v3.1 Vector:** AV:P/AC:L/PR:N/UI:N/S:U/C:M/I:N/A:N  
**CVSS Score:** 4.3

### Description
The complete HTML source code of the credential harvesting portal, including POST endpoint paths and form field names, is recoverable from the firmware binary via simple string extraction. Also, the binary also leaks the developer's local machine directory structure, which was embedded as a compiler artifact during the build.

### Findings Within This Vulnerability

**3a — Captive Portal Source Code Exposed**  
Location: Offset `0x10314`, `0x104B6` (binwalk), strings lines 288–345  
Recoverable data:
- Complete HTML page structure
- POST endpoint: `/capture`
- Form fields: `email`, `password`
- Apple captive portal intercept path: `/hotspot-detect.html`

**3b — Development Machine Path Leaked**  
Location: Offset `0x10CA0`, `0x1146D`, `0x11D3D`  
Recoverable data:
- Full path: `/home/kali/.arduino15/packages/esp32/hardware/esp32/3.3.8/...`
- Reveals: OS (Linux), username (kali), Arduino IDE config location, ESP32 core version

**3c — Anomalous User-Agent String**  
Hardcoded: `Mozilla/5.0 (Windows NT 5.1)...Chrome/30.0.1599.101`  
Impact: Unique device fingerprint — network analysts can identify this device from traffic alone. Windows XP + Chrome 30 user agent is instantly flagged by any modern IDS/SIEM.

### Impact
- Defender recovering device immediately knows its full attack capability
- Developer's machine and toolchain fingerprinted — enables targeted follow-up
- Device identifiable on network from its HTTP traffic signature alone

### Remediation
- Enable flash encryption — makes string extraction yield ciphertext
- Strip debug symbols and compiler path metadata via build flags
- Randomise or rotate User-Agent strings at runtime rather than hardcoding

---

## Summary Table

| ID | Title | Severity | CVSS | Remediation Effort |
|----|-------|---------|------|--------------------|
| VUL-01 | Hardcoded credentials in plaintext | Critical | 6.1 | Medium — requires provisioning redesign |
| VUL-02 | Unauthenticated UART bootloader | High | 6.8 | Low — single eFuse burn |
| VUL-03 | Source reconstructible from binary | Medium | 4.3 | Low — flash encryption |
| VUL-04 | No security eFuses burned | High | 6.8 | Low — eFuse provisioning |

---
# ⚠️ Disclaimer

*All testing conducted on personally owned hardware in an isolated home lab environment.*  
*No third-party systems were accessed or affected at any stage of this research.*
---