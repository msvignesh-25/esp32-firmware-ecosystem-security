----------------------------------------------------------------
STAGE 8 — DEFENCE IMPLEMENTATION & HARDENING GUIDE
ESP32 Firmware Security Analysis | Part 2
----------------------------------------------------------------

> Demonstratation of the available hardware-level defences on the ESP32, shows the effect of UART download mode restriction, and documentation of the complete hardening procedure for secure boot and flash encryption — with full explanation of irreversibility risks.

----------------------------------------------------------------

<span style="color:red">CRITICAL WARNING — READ BEFORE PROCEEDING</span>
------------------------------------------
ESP32 security hardening involves burning eFuse bits. eFuses are
one-time programmable hardware fuses. Once a bit is set:

  - It CANNOT be cleared by software
  - It CANNOT be cleared by reflashing
  - It CANNOT be cleared by erasing flash
  - It CANNOT be cleared by any means whatsoever
  - It is PERMANENT for the lifetime of the chip

A misconfigured secure boot key or flash encryption key will
result in a permanently bricked device. The chip cannot be
recovered by Espressif or any third party.

For this project: hardening commands are documented below but
were NOT executed. The UART
download mode behaviour was demonstrated using a safe simulation.

----------------------------------------------------------------

PART A — UART DOWNLOAD MODE RESTRICTION (DEMONSTRATED)
--------------------------------------------------------
This demonstration simulates the attacker experience when
attempting to connect to a hardened device.

SIMULATION METHOD:

  The --before no_reset flag instructs esptool to skip the
  bootloader trigger sequence (GPIO0 pull LOW + EN reset pulse).
  Without this trigger, the ESP32 remains in normal sketch
  execution mode and does not enter download mode.

Command:
  `esptool.py --port /dev/ttyUSB0 --baud 115200 --before no_reset read_flash 0x0 0x1000 test.bin`

Output observed:

  Serial port /dev/ttyUSB0<br>
  Note: Pre-connection option "no-reset" was selected.<br>
  Connection may fail if the chip is not in bootloader
  or flasher stub mode.<br>
  Connecting......................................<br>
  A fatal error occurred: Failed to connect to Espressif device:
  Invalid head of packet (0x9C): Possible serial noise or corruption.

EXPLANATION OF 0x9C ERROR:

  esptool sent its bootloader handshake sequence and waited for
  the ESP32 to respond with a bootloader acknowledgment byte.
  Instead, it received 0x9C — data being output by the running
  sketch on the UART line. The ESP32 was not in bootloader mode,
  so it responded with sketch serial output rather than a
  bootloader protocol response. esptool correctly identified this
  as an invalid response and aborted.

  On a device with UART_DOWNLOAD_DIS eFuse burned, this failure
  is permanent — even the correct GPIO0 + EN trigger sequence
  is ignored by the hardware. The ROM bootloader simply does not
  activate regardless of pin state.

COMPARISON TABLE:
  Scenario                          | Result
  ----------------------------------|--------------------------------
  Unprotected device, normal attack | Full 4MB extracted in 2 min
  Simulated hardening (no_reset)    | Fatal error — 0x9C — no data
  UART_DOWNLOAD_DIS burned (perm.)  | Fatal error — no response ever

----------------------------------------------------------------

PART B — DISABLE UART DOWNLOAD MODE (PERMANENT)
-------------------------------------------------
⚠️  <span style="color:red">IRREVERSIBLE — BURNS EFUSE — RUN AT YOUR OWN RISK</span>

Command:
  `espefuse.py --port /dev/ttyUSB0 burn_efuse UART_DOWNLOAD_DIS`

Effect:

  Sets UART_DOWNLOAD_DIS bit in BLOCK0 eFuse permanently.
  Valid on ESP32 v3 and newer (confirmed — this device is v3.1).
  After burning: esptool.py can never connect to this device again.
  The Stage 1 attack becomes permanently impossible.

eFuse state after burning:

  UART_DOWNLOAD_DIS = True R/W → True R/ (write disabled)

----------------------------------------------------------------

PART C — FLASH ENCRYPTION (PERMANENT)
---------------------------------------
⚠️  <span style="color:red">IRREVERSIBLE — BURNS EFUSE — LOSS OF KEY = PERMANENT BRICK</span>

Flash encryption uses AES-256 hardware acceleration built into
the ESP32. Once enabled, all flash read/write operations are
transparently encrypted/decrypted by hardware. An attacker
extracting the firmware receives only ciphertext. The strings
command yields no readable output. Ghidra analysis shows only
encrypted data.

Step 1 — Generate flash encryption key:
  `espsecure.py generate_flash_encryption_key flash_encryption_key.bin`

  ⚠️  **Back up flash_encryption_key.bin securely. If lost, the
  device can never be reflashed. Store in an encrypted vault.**

Step 2 — Burn key to eFuse BLOCK1:
  `espefuse.py --port /dev/ttyUSB0 burn_key BLOCK1 flash_encryption_key.bin FLASH_ENCRYPTION`

Step 3 — Enable flash encryption by either **ESP-IDF** *menuconfig* interface or by manually adding `CONFIG_FLASH_ENCRYPTION_ENABLED=y` to the sdkconfig file. During the next boot cycle, the ESP32 will automatically encrypt all of its flashed contents.

eFuse state after enabling:

  FLASH_CRYPT_CNT = odd number of bits set (encryption active)
  BLOCK1          = AES-256 key (32 bytes, no longer readable)

Effect on attack:

  Stage 1 : esptool still reads flash — but output is ciphertext
  Stage 4 : strings finds no readable credentials
  Stage 5 : Ghidra shows encrypted binary — no string analysis
  Result  : Firmware extraction attack chain is completely broken

----------------------------------------------------------------

PART D — SECURE BOOT V2 (PERMANENT)
--------------------------------------
⚠️  <span style="color:red">IRREVERSIBLE — BURNS EFUSE — WRONG KEY = PERMANENT BRICK</span>

Secure boot ensures the ESP32 only executes firmware images signed
with a specific private key. Unsigned or tampered firmware is
rejected by the ROM bootloader before execution begins.

Step 1 — Generate RSA-3072 or ECDSA-P256 signing key:
    `espsecure.py generate_signing_key --version 2 secure_boot_signing_key.pem`

  ⚠️  **This private key must be stored securely. If lost, no new
  firmware can ever be signed for this device. Device becomes
  permanently locked to the last flashed firmware version.**

Step 2 — Sign the firmware image:
  `espsecure.py sign_data --version 2 --keyfile secure_boot_signing_key.pem --output firmware_signed.bin firmware.bin`

Step 3 — Burn public key digest to eFuse BLOCK2:
    `espefuse.py --port /dev/ttyUSB0 burn_key BLOCK2 secure_boot_signing_key.pem SECURE_BOOT_V2`

Step 4 — Flash signed firmware and enable secure boot:
  `esptool.py --port /dev/ttyUSB0 write_flash 0x0 firmware_signed.bin`

eFuse state after enabling:

  ABS_DONE_1 = True (Secure boot V2 permanently active)
  BLOCK2     = Public key digest (32 bytes, write protected)

Effect on attack:

  Firmware replacement attack : Rejected at boot — signature invalid
  Modified firmware           : Rejected at boot — digest mismatch
  Result                      : Device integrity permanently protected

----------------------------------------------------------------

PART E — DISABLE JTAG (PERMANENT)
-----------------------------------
⚠️  <span style="color:red">IRREVERSIBLE — disables hardware debug interface permanently</span>

Command:
  `espefuse.py --port /dev/ttyUSB0 burn_efuse JTAG_DISABLE`

Effect:

JTAG debugging interface permanently disabled. Prevents
hardware-level memory inspection and single-step debugging attacks
that could bypass software-level security measures.

----------------------------------------------------------------

BEFORE / AFTER SUMMARY
------------------------
Attack                  | Before Hardening    | After Full Hardening
------------------------|---------------------|----------------------
esptool read_flash      | Full dump in 2 min  | UART blocked forever
strings — credentials   | Plaintext visible   | Ciphertext only
Ghidra analysis         | Strings readable    | Encrypted — useless
Bootloader flash_id     | Hardware info given | No response
JTAG debugging          | Fully accessible    | Permanently blocked
Tampered firmware boot  | Accepted            | Rejected at ROM level
Firmware downgrade      | Possible            | Blocked by anti-rollback

----------------------------------------------------------------

PRODUCTION SECURITY REQUIREMENTS
-------------------------------------------------------
If you are deploying an ESP32 or any embedded IoT device that
handles sensitive credentials, user data, or network access
in a production environment, consider the following as the baseline checklist:

1. NEVER hardcode credentials in firmware source code.
   Use provisioning workflows, encrypted NVS partitions, or
   secure element instead. Credentials in source become
   credentials in binary, as demonstrated

2. Burn the *UART_DOWNLOAD_DIS* eFuse to close an unauthenticated root access channel on shipped devices.

3. Secure boot ensures ESP32 only executes signed, trusted firmware, while flash encryption protects the data at rest. 

4. Disable JTAG: Don't leave hardware debug interfaces active on deployed devices. It's also an attack surface.

Security by obscurity — assuming attackers will not analyse your
binary, will not have physical access, or will not know about
esptool — is not a defence strategy. The tools used to extract and reverse-engineer the firmware are free, open-source, pre-installed on Kali Linux.
The only barrier to entry is a USB cable.

Hardware-level protections (eFuse burns) are the only reliable defenses. These are zero-cost configurations that prevent any firmware compromise

---
### ⚠️ Disclaimer
This project was conducted on personally owned hardware in a
controlled home lab environment for security research and
educational purposes only.