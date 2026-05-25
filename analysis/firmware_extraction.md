----------------------------------------------------------------
STAGE 1 — FIRMWARE EXTRACTION
ESP32 Firmware Security Analysis | Part 2
----------------------------------------------------------------

> Extraction of complete flash contents of the ESP32 device using `esptool.py` over the UART bootloader interface. No hardware modification required. Physical USB access is the only requirement.

----------------------------------------------------------------

TARGET DEVICE
-------------
```
ESP32-D0WD-V3 (revision v3.1)
Flash chip manufacturer : 68 (ISSI / compatible)
Flash chip device ID    : 4016
Flash size              : 4MB
Flash voltage           : 3.3V (determined by strapping pin)
CPU architecture        : Xtensa LX6 dual-core 32-bit

```
Refer [flash memory information](../assets/stage6_flash-id.png)

----------------------------------------------------------------

EXTRACTION COMMAND
------------------
### No sudo required for port access

`esptool.py --port /dev/ttyUSB0 --baud 115200 read_flash 0x0 ALL firmware.bin`

----------------------------------------------------------------

HOW ESPTOOL CONNECTS TO THE BOOTLOADER
---------------------------------------
esptool.py triggers the ESP32 ROM bootloader using a hardware handshake sequence over the UART lines:

  1. Pulls GPIO0 LOW via RTS/DTR pin on the USB-UART bridge
  2. Pulses the EN (reset) pin LOW then HIGH
  3. ESP32 boots into download mode instead of running the sketch
  4. esptool establishes SLIP (Serial Line Internet Protocol) session
  5. Sends read_flash command — ESP32 streams flash contents over UART
  6. esptool writes received bytes sequentially to firmware.bin

No authentication is involved at any stage. The ROM bootloader accepts commands from any host that correctly triggers this sequence.

----------------------------------------------------------------

SECURITY IMPLICATION
--------------------
The entire flash memory — bootloader, partition table, application
firmware, file system, NVS storage region — was extracted without:

  - Any PIN or password
  - Any cryptographic challenge
  - Any hardware modification
  - Any specialised equipment beyond a standard USB cable

Total time from device connection to complete firmware dump: under 2 minutes. This is the foundational attack that all subsequent stages build upon.

----------------------------------------------------------------

FILES PRODUCED
--------------
firmware.bin — raw 4MB flash dump, used in all subsequent stages