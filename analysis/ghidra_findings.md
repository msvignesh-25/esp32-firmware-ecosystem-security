----------------------------------------------------------------
STAGE 5 — GHIDRA REVERSE ENGINEERING
ESP32 Firmware Security Analysis | Part 2
----------------------------------------------------------------

> Importing the extracted firmware binary into Ghidra and perform static analysis — locating credential strings at the assembly level,identifying the function responsible for WiFi initialisation, and confirming that plaintext credentials are directly referenced in disassembly without any decryption layer.

----------------------------------------------------------------

PROJECT SETUP
-------------
```
Settings applied:
  Format   : Raw Binary
  Language : Xtensa:LE:32:default
  (Xtensa is the CPU architecture inside the ESP32 — LX6 core, little-endian, 32-bit. Selecting the correct architecture is mandatory for correct disassembly.)
```

Analysis duration: approximately 3-5 minutes.

----------------------------------------------------------------

ARCHITECTURE NOTE — XTENSA
---------------------------
The ESP32 uses a Tensilica Xtensa LX6 processor — not x86, ARM, or MIPS. Ghidra supports Xtensa through its built-in processor module, but decompilation quality is lower than for mainstream architectures. The disassembly (middle panel) is reliable.

----------------------------------------------------------------

SYMBOL TREE OBSERVATION
------------------------
After analysis, the Symbol Tree (left panel) showed three folders:

  .text    — executable code sections (compiled sketch functions)<br>
  .rodata  — read-only data (string literals, constants, tables)<br>
  External — library function references

Function name search attempts:

  Filter: "setup"   → No results<br>
  Filter: "wifi"    → No results<br>
  Filter: "capture" → No results<br>

Reason: Debug symbols are stripped from Arduino-compiled ESP32 firmware by default during the linking stage. All user-defined function names (setup, loop, handleCapture, etc.) are replaced with auto-generated labels such as FUN_002004c8, FUN_001ff2a4.

This is standard behaviour for production firmware.

----------------------------------------------------------------

PRIMARY FINDING — SSID LOCATED AT 5 MEMORY ADDRESSES
------------------------------------------------------
Method: Search → Memory → search string "Airtel" (actual SSID)

Results: 5 entries found

  Location    | Match Bytes            | Match Value
  ------------|------------------------|------------
  0x00009944  | 41 69 72 74 ...        | Airtel
  0x00009d64  | 41 69 72 74 ...        | Airtel
  0x0000b224  | 41 69 72 74 ...        | Airtel
  0x0000b524  | 41 69 72 74 ...        | Airtel
  0x00010a92  | 41 69 72 74 ...        | Airtel

- Refer [SSID Reveal](../assets/Stage5_SSID.png)

Byte sequence confirmed: 41 69 72 74  = ASCII "AIRTEL"
(case-insensitive match — actual stored bytes are lowercase "airtel")

All 5 locations double-clicked individually. All 5 references
resolved to a same function in the disassembly

----------------------------------------------------------------

FUNCTION ANALYSIS
----------------------------------
This auto-named function is the compiled equivalent of the
Arduino sketch's setup() function. The compiler retained all
string literal content but stripped the function name.

Why the same function appears 5 times:

  The SSID string is used in multiple contexts within setup():

    1. WiFi.begin(ssid, password) call — station mode connection
    2. WiFi.softAP() configuration — access point mode SSID
    3. NVS write operation — credential storage to flash
    4. Serial.println() debug output — logged to UART console
    5. DNS/captive portal configuration reference

  The compiler either stores one string and references it 5 times
  from the same function, or copies it to multiple memory locations
  for different subsystem uses. Both result in 5 hits pointing back
  to the same parent function.

----------------------------------------------------------------

DISASSEMBLY OBSERVATION
------------------------
After double-clicking any of the 5 SSID memory addresses, Ghidra
jumped to the corresponding location in the middle disassembly panel.

Observation: Both the **SSID** and **password** strings were directly
visible as string literals in the disassembly view — the CPU
loads memory pointers to these raw strings into Xtensa registers
(a3, a4) before passing them to the WiFi connect call.

Xtensa assembly pattern observed (representative):
  l32r    a3, DAT_00009944    ; load pointer to SSID string
  l32r    a4, DAT_0000xxxx    ; load pointer to password string
  call8   FUN_xxxxxxxx        ; call WiFi.begin(ssid, password)

No decryption function is called before the strings are loaded.
No transformation is applied to the credential values. The raw
plaintext strings are passed directly to the WiFi connect function.

----------------------------------------------------------------

SUMMARY
-------
Ghidra confirmed at binary/assembly level what strings analysis
found at the text level:

  - SSID string physically present at 5 memory addresses
  - All references originate from one compiled function (setup())
  - Password string located adjacent to SSID in memory
  - Both strings passed as raw pointers directly to WiFi connect
  - No encryption, encoding, or obfuscation layer exists between
    the stored credential and its use in the WiFi connection call

This constitutes assembly-level proof of the 1st VULN finding
documented in the vulnerability report.

----------------------------------------------------------------

NOTE
--------------------
Ghidra's Xtensa support, while functional for disassembly, has
lower decompiler fidelity compared to x86 or ARM targets. For
deeper function-level analysis of ESP32 firmware, IDA Pro with
the Xtensa processor module or Binary Ninja with community plugins
would provide better decompiler output. For the purposes of this
project — locating and confirming credential storage at assembly
level — Ghidra was fully adequate.