----------------------------------------------------------------
STAGE 3 — BINARY ANALYSIS WITH BINWALK
ESP32 Firmware Security Analysis | Part 2
----------------------------------------------------------------

> Scanning of the extracted firmware.bin using `binwalk` to identify embedded file signatures, compressed sections, partition structure, and any recognisable data formats present in the binary.

----------------------------------------------------------------

FULL OUTPUT
----------------------------
```

DECIMAL       HEXADECIMAL     DESCRIPTION
4096          0x1000          ESP Image (ESP32): segment count: 3,
                              flash mode: DIO, flash speed: 80MHz,
                              flash size: 4MB,
                              entry address: 0x400805b4, hash: sha256
32768         0x8000          ESP32 Partition Table Entry:
                              label: "nvs", type: DATA,
                              subtype: NVS, offset: 0x9000,
                              size: 0x5000, flags: 0x0 (not encrypted)
32800         0x8020          ESP32 Partition Table Entry:
                              label: "otadata", type: DATA,
                              subtype: Factory/OTA DATA,
                              offset: 0xe000, size: 0x2000
32832         0x8040          ESP32 Partition Table Entry:
                              label: "app0", type: APP,
                              subtype: OTA 0, offset: 0x10000,
                              size: 0x140000, flags: 0x0
32864         0x8060          ESP32 Partition Table Entry:
                              label: "app1", type: APP,
                              subtype: OTA 1, offset: 0x150000,
                              size: 0x140000, flags: 0x0
32896         0x8080          ESP32 Partition Table Entry:
                              label: "spiffs", type: DATA,
                              subtype: SPIFFS, offset: 0x290000,
                              size: 0x160000, flags: 0x0
32928         0x80A0          ESP32 Partition Table Entry:
                              label: "coredump", type: DATA,
                              subtype: Coredump, offset: 0x3f0000,
                              size: 0x10000, flags: 0x0
65536         0x10000         ESP Image (ESP32): segment count: 6,
                              flash mode: DIO, flash speed: 80MHz,
                              flash size: 4MB,
                              entry address: 0x40081eb0, hash: sha256
66324         0x10314         HTML document header
66716         0x1049C         HTML document footer
66742         0x104B6         HTML document header
68211         0x10A73         HTML document footer
68768         0x10CA0         Unix path:
                              /home/kali/.arduino15/packages/esp32/
                              hardware/esp32/3.3.8/libraries/
                              WebServer/src/Parsing.cpp
70765         0x1146D         Unix path:
                              /home/kali/.arduino15/packages/esp32/
                              hardware/esp32/3.3.8/cores/esp32/
                              esp32-hal-uart.c
73021         0x11D3D         Unix path:
                              /home/kali/.arduino15/packages/esp32/
                              tools/esp32-libs/3.3.8/include/hal/
                              esp32/include/hal/timer_ll.h
161748        0x277D4         SHA256 hash constants, little endian
163144        0x27D48         AES Inverse S-Box
163656        0x27F48         AES S-Box
520424        0x7F0E8         ELF, 32-bit LSB processor-specific
1048576       0x100000        ESP Image (ESP32): segment count: 7,
                              flash mode: DIO, flash speed: 40MHz,
                              flash size: 4MB,
                              entry address: 0x40080ea8, hash: sha256
1161380       0x11B8A4        PEM certificate
1162572       0x11BD4C        SHA256 hash constants, little endian
1162908       0x11BE9C        PEM RSA private key
1162972       0x11BEDC        PEM EC private key
1163032       0x11BF18        PEM PKCS#8 private key
```
Refer [full binwalk output](../assets/stage3_binwalk-full.png)

----------------------------------------------------------------

PARTITION TABLE ANALYSIS
-------------------------
The complete flash layout was reconstructed from the binary:

Partition  | Offset     | Size    | Purpose
-----------|------------|---------|---------------------------
nvs        | 0x9000     | 20 KB   | Runtime WiFi credential store
otadata    | 0xe000     | 8 KB    | OTA update state tracking
app0       | 0x10000    | 1280 KB | Main application (sketch)
app1       | 0x150000   | 1280 KB | OTA backup slot
spiffs     | 0x290000   | 1408 KB | File system — HTML, configs
coredump   | 0x3F0000   | 64 KB   | Crash dump storage

Key observation: app0 at 0x10000 contains the compiled sketch
with all hardcoded strings — this is the primary analysis target
for Stages 4 and 5.

----------------------------------------------------------------

Key Findings
---------------------------

1. HTML document headers at 0x10314 and 0x104B6
   The captive portal HTML page is embedded as plaintext inside
   the app0 partition. Full source is recoverable — see Stage 4.

2. Unix paths at 0x10CA0, 0x1146D, 0x11D3D
   The full filesystem path of the development machine is leaked
   as a compiler debug artifact. Reveals: OS (Linux), username
   (kali), Arduino IDE version, ESP32 core version (3.3.8).

3. AES S-Box and SHA256 constants
   These are mbedTLS cryptographic library lookup tables compiled
   into the firmware for WPA2 WiFi authentication. Not keys —
   these are algorithm constants present in every ESP32 firmware
   that uses WiFi.

4. PEM markers at 0x11B8A4 onwards
   binwalk flagged PEM certificate and private key headers. Manual
   verification via strings confirmed these are mbedTLS parser
   pattern strings — not actual keys. The firmware uses HTTP only,
   so no real TLS certificates are present. This demonstrates that
   binwalk findings require manual verification before reporting.

5. ELF binary at 0x7F0E8
   A 32-bit LSB ELF segment embedded within the firmware image —
   part of the compiled application binary format used by the
   ESP32 toolchain.

----------------------------------------------------------------

### Conclusion
- `binwalk` confirmed the firmware is a valid, complete ESP32 flash
dump containing an active application with embedded HTML, leaked
development paths, and standard cryptographic library components.
The partition table is fully mapped and consistent with a standard
Arduino-generated ESP32 sketch with OTA support enabled.