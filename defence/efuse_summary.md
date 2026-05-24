----------------------------------------------------------------
STAGE 6 — UART BOOTLOADER ACCESS & EFUSE SECURITY STATE
ESP32 Firmware Security Analysis | Part 2
----------------------------------------------------------------

> Accessing the ESP32 ROM bootloader via UART to demonstrate what an attacker with physical access can do unauthenticated, and read the complete eFuse security configuration to establish the baseline security state of the device before any hardening is applied.

----------------------------------------------------------------

PART A — BOOTLOADER IDENTIFICATION
------------------------------------
Command:
`esptool.py --port /dev/ttyUSB0 --baud 115200 flash_id`

Output:

  Detecting chip type... ESP32<br>
  Chip is ESP32-D0WD-V3<br>
  Features: WiFi, BT, Dual Core, 240MHz, VRef calibration in efuse, Coding Scheme: None<br>
  Crystal is 40MHz<br>
  MAC: [REDACTED]<br>
  Manufacturer: 68<br>
  Device: 4016<br>
  Detected flash size: 4MB<br>
  Flash voltage (determined by strapping pin): 3.3V<br>

Analysis:
  The ROM bootloader returned complete hardware identification with
  zero authentication. An attacker learns:
    - Exact chip revision (ESP32-D0WD-V3 rev 3.1)
    - Feature set (dual core, Bluetooth, WiFi)
    - Crystal frequency (40MHz)
    - MAC address (hardware identifier — privacy concern)
    - Flash manufacturer and device ID
    - Flash capacity and operating voltage

  This information enables precise targeting of known chip-specific
  vulnerabilities and confirms which attack tools and parameters
  to use for subsequent flash operations.

----------------------------------------------------------------

PART B — WHAT THE BOOTLOADER CHANNEL ALLOWS
---------------------------------------------
The UART download mode bootloader, once triggered, accepts the
following commands with no further authentication:

  read_flash   — Read any region of flash memory
  write_flash  — Write new firmware to any flash address
  erase_flash  — Erase entire flash contents
  erase_region — Erase a specific flash region
  read_mem     — Read from memory-mapped addresses
  write_mem    — Write to memory-mapped addresses
  flash_id     — Read flash chip identification

An attacker with physical USB access and esptool can:
  1. Extract entire firmware (demonstrated — Stage 1)
  2. Replace firmware with malicious version
  3. Erase flash entirely — destroying all evidence
  4. Dump specific partitions (NVS, SPIFFS, coredump)
  5. Read runtime memory values via read_mem

All of the above require only a USB cable and a laptop.

----------------------------------------------------------------

PART C — RASPBERRY PI AS ATTACK PLATFORM
------------------------------------------
The complete attack chain — esptool, binwalk, strings — runs
identically on a Raspberry Pi Zero W (a thumb-sized device) running Raspberry Pi OS
(Debian-based, same package ecosystem as Kali).

Implication: An attacker can conceal a Pi Zero W inside a target
environment — a server room, equipment cabinet, or office space —
connected to a target ESP32 device. The Pi connects over WiFi,
receives commands remotely, and executes the full attack chain
without the attacker being physically present at the time of
extraction. This makes the threat model significantly more
practical in real-world scenarios.

----------------------------------------------------------------

PART D — EFUSE SECURITY STATE (BASELINE — BEFORE HARDENING)
--------------------------------------------------------------
Command:
  `espefuse.py --port /dev/ttyUSB0 summary`

The eFuse (Electrically Fusible) memory in the ESP32 stores
one-time programmable security configuration. Once a bit is
written (burned), it cannot be reset — not by reflashing,
not by erasing, not by any software means.

SECURITY FUSES — COMPLETE STATE
---------------------------------
eFuse Name            | Value  | Meaning
----------------------|--------|----------------------------------
FLASH_CRYPT_CNT       | 0      | Flash encryption DISABLED
FLASH_CRYPT_CONFIG    | 0      | No encryption key config
ABS_DONE_0            | False  | Secure boot V1 DISABLED
ABS_DONE_1            | False  | Secure boot V2 DISABLED
UART_DOWNLOAD_DIS     | False  | UART download mode ENABLED
JTAG_DISABLE          | False  | JTAG debugging ENABLED
DISABLE_DL_ENCRYPT    | False  | DL mode encryption not disabled
DISABLE_DL_DECRYPT    | False  | DL mode decryption not disabled
KEY_STATUS            | False  | eFuse block 3 not reserved
SECURE_VERSION        | 0      | Anti-rollback not configured

KEY STORAGE STATE
------------------
eFuse Block | Purpose              | Contents
------------|----------------------|---------------------------
BLOCK1      | Flash encryption key | 00 00 00 00 ... (32 bytes)
BLOCK2      | Secure boot key      | 00 00 00 00 ... (32 bytes)
BLOCK3      | Variable / custom    | 00 00 00 00 ... (32 bytes)

All key blocks are zero — no cryptographic keys have been
provisioned. This is the factory default state.

OTHER NOTABLE FUSES
--------------------
eFuse Name            | Value              | Note
----------------------|--------------------|----------------------
CHIP_VER_REV1         | True               | Silicon revision 1
CHIP_VER_REV2         | True               | Silicon revision 2
WAFER_VERSION_MAJOR   | 3                  | Chip version 3
CONSOLE_DEBUG_DISABLE | True               | ROM BASIC disabled
CHIP_CPU_FREQ_RATED   | True               | Rated for 240MHz
ADC_VREF              | 1093               | ADC calibration value
MAC                   | [REDACTED]         | Hardware MAC address
CODING_SCHEME         | NONE               | BLK1-3 = 256 bits each

----------------------------------------------------------------

SECURITY ASSESSMENT SUMMARY
-----------------------------
Attack vector          | Status
-----------------------|----------------------------------------
UART firmware dump     | Fully open — no restriction
UART firmware replace  | Fully open — no restriction
JTAG debug access      | Fully open — not disabled
Flash content readable | Fully open — no encryption
Unsigned firmware boot | Accepted — no secure boot
Anti-rollback          | Not configured — downgrade possible

This device is in factory default security state. Every hardware
attack vector is available to any attacker with physical access
and a USB cable. No eFuse-based protection has been applied.

This establishes the "before" baseline. The hardening guide in
defence/hardening_guide.txt documents the steps required to
close each of these attack vectors.