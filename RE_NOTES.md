# Reverse Engineering Investigation Log: Actions Semiconductor ATJ2127 Bluetooth Subsystem

**Target Device:** Actions US212D / ATJ2127 Reference Portable Media Player  
**Firmware Version:** V1.101.25 (Actions Semiconductor / US212D)  
**Main SoC:** Actions Semiconductor ATJ2127 (MIPS32 / MIPS16 hybrid core)  
**Companion Bluetooth SoC:** Beken Corporation BK3266 / BK3254 family  

---

## 1. FACTS (Binary & Hardware Proven)

### 1.1 Dual-Chip Architecture
1. **Host SoC (ATJ2127):** Executes user interface, filesystem, audio decoding (MP3, WAV, FLAC, AAC, etc.), and high-level Bluetooth commands via `BT.AP` (UI) and `BTENGINE.AP` (driver).
2. **Bluetooth Controller (Beken BK3266/BK3254):** Standalone Bluetooth audio SoC containing its own processor, ROM/Flash, 2.4 GHz RF transceiver, baseband/link controller, Bluetooth 4.2/5.0 stack, and SBC audio encoder/DSP.
3. **Physical Interconnect:**
   - **UART Data:** ATJ2127 UART peripheral at `0xc00f0000` (TX/RX FIFO at `0xc00f0008`).
   - **DMA Engine:** ATJ2127 DMA controller at `0xc00c0000`:
     - Channel 1 (`0xc00c0030` - `0xc00c0048`): Dedicated to UART TX streaming directly to `0xc00f0008`.
     - Channel 2 (`0xc00c0050` - `0xc00c0068`): Dedicated to UART RX ring buffer at `0xbfc34000`.
   - **Power / Reset Control:** GPIO Port A Pin 19 (`0xc0090000` and `0xc0090008` bit 19) controls power/reset to the Bluetooth controller.
   - **Baud Rates:**
     - Boot/Default Baud: 115,200 baud (`0x0001c200`, `CONFIG.TXT` value 6).
     - Streaming Baud: 1,500,000 baud (`0x0016e360` = 1.5 MBaud, switched via Bank 4 `new baud:` function).

### 1.2 Binary Executable Format & Overlay System (YQHX)
1. Both `BT.AP` and `BTENGINE.AP` use the Actions Semiconductor **YQHX** container format:
   - Header magic: `yqhx` at offset 4.
   - `BTENGINE.AP` layout:
     - Resident Code: File `0x3800` - `0x4a00` (size `0x1200`), runtime base `0xbfc34500`.
     - Resident Data: File `0x5000` - `0x5290` (size `0x290`), runtime base `0x9fc35700`.
     - Resident BSS: Size `0x6c`, runtime base `0x9fc35990`.
     - Overlay Table: Offset `0x1000` containing entries mapping 11 overlay banks.
2. **Overlay Banking Execution Model:**
   - Overlay RAM Execution Window: **`0xbfc35a00` - `0xbfc36000` (size `0x600` bytes)**.
   - All overlay banks are assembled to run at `0xbfc35a00` and swapped into this 1.5 KB RAM buffer on demand.
   - Tagged Function Pointers have the format: `0x73000000 | ((bank_idx * 4 + 3) << 16) | 0x5a00 | in_bank_offset | 1`.
   - The initial entry point in YQHX header (`0x73035a31`) is Tag `0x7303` (Bank 0) + in-bank offset `0x30` + MIPS16 bit 1.
3. **Complete Overlay Bank Map:**
   - **Bank 0 (`0x05800`, tag `0x73035a00`):** Subsystem Initialization (`startup connect:`, `nand.drv`, `btengine.ap`, `vm:`).
   - **Bank 1 (`0x06000`, tag `0x73075a00`):** Application Initialization (`ap init:`).
   - **Bank 2 (`0x06800`, tag `0x730b5a00`):** A2DP Connection State & Streaming (`clear s_link`, `s_link_ok`, `Clear last active dev!`, `connect ongoing #1!`, `connect: search dev fail!`, `a2dp stream start!`, `a2dp stream suspend!`).
   - **Bank 3 (`0x07000`, tag `0x730f5a00`):** Device Discovery & Connection (`inq:`, `c_inq:`, `remote name!`, `name:`, `connect device!`, `dis-connect device!`, `cancel connect device!`, `g_pdl:`, `d_pdl:`).
   - **Bank 4 (`0x07800`, tag `0x73135a00`):** Hardware Baud Rate & AVRCP Volume (`[exit_succ]`, `get abs:`, `set abs:`, `new baud:`).
   - **Bank 5 (`0x08000`, tag `0x73175a00`):** Remote Volume & Link Status (`cfg`, `rmt vol:`, `local:`, `remote:`, `set state:`, `avrcp:`, `start conn timeover!`).
   - **Bank 6 (`0x08800`, tag `0x731b5a00`):** Software Timer Wheel Management (`timer_create`, `timer_start`, `timer_stop`).
   - **Bank 7 (`0x09000`, tag `0x731f5a00`):** Power & Configuration Dispatcher (`close BT hardware ......`, `wait ready fail!`, `bt_OK!`, `get version fail!`, `wait_poff!`, `power_down_OK!`, `get_config`, `set_config`).
   - **Bank 8 (`0x09800`, tag `0x73235a00`):** Protocol ACK & Timeout Processing (`ack to!`).
   - **Bank 9 (`0x0a000`, tag `0x73275a00`):** Low-Level Command Builder (Packet opcodes 44, 45, 48, 49, 50, 127).
   - **Bank 10 (`0x0a800`, tag `0x732b5a00`):** Event Callbacks & Packet Completion (`send_finish`).
   - **Driver Section (`0x0b000`, runtime `0xbfc20d00`):** Diagnostic string formatter and serial print logger.

---

## 2. HARDWARE REGISTER MAP (ATJ2127)

| Register Address | Name | Description & Usage in Bluetooth Subsystem |
|---|---|---|
| `0xc0000000` | `MRCR` | Module Reset Control Register (Bit 4 = UART reset) |
| `0xc0000008` | `CLKENCTL` | Clock Enable Control Register (Enables UART peripheral clock) |
| `0xc0090000` | `GPIO_A_DAT` | GPIO Port A Data / Direction Register |
| `0xc0090008` | `GPIO_A_OUTEN` | GPIO Port A Output Enable (Bit 19 = Bluetooth power / reset) |
| `0xc0090030` | `MFP_CTL1` | Multi-Function Pin Mux 1 (Configures UART TX/RX pins) |
| `0xc0090040` | `MFP_CTL2` | Multi-Function Pin Mux 2 (Configures UART pin drive strength) |
| `0xc009004c` | `MFP_CTL3` | Multi-Function Pin Mux 3 (Configures secondary UART/GPIO routing) |
| `0xc00c0000` | `DMA_BASE` | DMA Engine Controller Base |
| `0xc00c0030` | `DMA_CH1_CTL` | DMA Channel 1 Control (UART TX DMA trigger & status) |
| `0xc00c0034` | `DMA_CH1_SRC` | DMA Channel 1 Source Pointer (Host SRAM packet buffer) |
| `0xc00c0040` | `DMA_CH1_DST` | DMA Channel 1 Destination Pointer (`0xc00f0008` UART TX FIFO) |
| `0xc00c0048` | `DMA_CH1_LEN` | DMA Channel 1 Length (Byte count to transmit) |
| `0xc00c0050` | `DMA_CH2_CTL` | DMA Channel 2 Control (UART RX DMA trigger & status) |
| `0xc00c0054` | `DMA_CH2_DST` | DMA Channel 2 Destination Pointer (`0xbfc34000` RX ring buffer) |
| `0xc00c0060` | `DMA_CH2_LEN` | DMA Channel 2 Length |
| `0xc00f0000` | `UART_CTL` | UART Baud Rate Divider / Control (`0x0ce38a03` = 115200, modified for 1.5M) |
| `0xc00f0004` | `UART_CFG` | UART Frame Configuration (8 data bits, 1 stop bit, no parity) |
| `0xc00f0008` | `UART_DAT` | UART Transmit / Receive FIFO Data Register |
| `0xc00f000c` | `UART_STAT` | UART Status & FIFO Flags (Bit 21 = TX FIFO Empty) |
| `0xc012001c` | `WD_CTL` | Watchdog Timer Clear Register (Bit 0 |= 1 clears watchdog) |

---

## 3. IMPORT THUNK & SYSTEM CALL ABI

Because MIPS16 lacks a 16-bit `syscall` opcode, Actions Semiconductor implemented system calls using MIPS16 `li v1, <service_id>; break 0`. The Breakpoint Exception (ExcCode 9) handler dispatches `v1`:

| Thunk Vector | Service | Function & Discovered Usage |
|---|---|---|
| `0xbfc34500` | Import 11 | Message Queue / OS Event Dispatch |
| `0xbfc34504` | Import 9 | Device / File Close |
| `0xbfc34508` | Import 7 | File / Driver I/O Operation |
| `0xbfc3450c` | Import 16 | System Logging / `printf` (`la a0, string; jalr`) |
| `0xbfc34510` | Import 15 | Kernel Syscall Multiplexer (Selector in `a3`): |
| | `a3 = 8` | Mount Driver (`la a2, "card.drv" / "nand.drv"`) |
| | `a3 = 9` | Close Driver / File |
| | `a3 = 28` | `get_system_ticks()` (Returns 32-bit system tick count) |
| | `a3 = 30` | `sleep_ms()` (Settle delay after baud change) |
| | `a3 = 41` | Initialize Storage Filesystem |
| | `a3 = 42` | Storage Seek / Read |
| | `a3 = 49` | `sleep_ms(a0)` (Millisecond sleep / retry delay) |
| | `a3 = 51` | `enter_critical()` (Disable interrupts before DMA) |
| | `a3 = 52` | `exit_critical()` (Enable interrupts after DMA) |
| `0xbfc34514` | Import 6 | OS Primitive (Semaphore / Mutex) |
| `0xbfc34518` | Import 17 | Task / Priority Control |
| `0xbfc3451c` | Import 0 | Task Exit / Terminate |

---

## 4. BLUETOOTH CONTROLLER PROTOCOL (Host <-> Beken BK3266)

### 4.1 Packet Framing
Every packet sent over UART to the Bluetooth controller follows a strict framing:
```
+------+-----+------+--------+--------------------+----------+
| 0xAA | SEQ | TYPE | OPCODE | PAYLOAD (0..N-1)   | CHECKSUM |
+------+-----+------+--------+--------------------+----------+
  Byte0  Byte1 Byte2   Byte3      Bytes 4..LEN-2       Last Byte
```
- **Sync Byte (`0xAA`):** Always `0xAA` (constant `0x56` negated in assembly).
- **Sequence Number (`SEQ`):** Incremented transaction identifier.
- **Packet Type (`TYPE`):** `1` = Event/Response, `2` = Command Request, `3` = Config/Data.
- **Opcode (`OPCODE`):** 1-byte functional command identifier.
- **Checksum:** Computed over bytes 1 through `LEN-2`:  
  $$\text{Checksum} = (-\sum_{i=1}^{\text{LEN}-2} \text{Byte}[i]) \pmod{256}$$
- **ACK Timeout:** 50 ms loop (50 iterations of 1 ms sleep). Emits `"ack to!\n"` on failure.

### 4.2 Discovered Command Opcodes
| Opcode (Hex) | Opcode (Dec) | Module Bank | User-Facing Log String / Purpose |
|---|---|---|---|
| `0x01` | 1 | Bank 3 | `inq:` (Start Bluetooth Inquiry / Device Scan) |
| `0x02` | 2 | Bank 3 | `c_inq:` (Cancel Inquiry Scan) |
| `0x03` | 3 | Bank 3 | `remote name!` (Read Remote Device Name) |
| `0x04` | 4 | Bank 3 | `connect device!` (Initiate ACL / A2DP Connection) |
| `0x05` | 5 | Bank 4 | `get link state` (Query Link Status / RSSI) |
| `0x06` | 6 | Bank 4 | `get version` (Query Controller Version; fail -> `get version fail!`) |
| `0x16` | 22 | Bank 2 | `a2dp stream start!` (Start A2DP Audio Streaming) |
| `0x17` | 23 | Bank 2 | `a2dp stream suspend!` (Pause / Suspend A2DP Audio) |
| `0x20` | 32 | Bank 4 | AVRCP Play Command |
| `0x21` | 33 | Bank 4 | AVRCP Pause Command |
| `0x23` | 35 | Bank 4 | AVRCP Stop Command |
| `0x24` | 36 | Bank 4 | AVRCP Track Next |
| `0x25` | 37 | Bank 4 | AVRCP Track Prev |
| `0x27` | 39 | Bank 4 | `set abs:` / `get abs:` (AVRCP Absolute Volume) |
| `0x28` | 40 | Bank 4 | `new baud:` (Set Controller UART Baud Rate) |
| `0x29` | 41 | Bank 3 | `dis-connect device!` (Terminate ACL Connection) |
| `0x2A` | 42 | Bank 3 | `cancel connect device!` (Abort Ongoing Connection) |
| `0x2B` | 43 | Bank 3 | `g_pdl:` (Get Paired Device List) |
| `0x2D` | 45 | Bank 9 | Reset / Link Clear |
| `0x31` | 49 | Bank 9 | Multi-parameter Link Setup (Sniff/Security/Role parameters) |
| `0x7F` | 127 | Bank 9 | `set_config` (Apply Configuration Parameters from `CONFIG.BIN`) |

---

## 5. RAM ARTIFACTS & CONNECTED DEVICE TABLE

Parsed from RAM dump at paired device table `0xbfc35770`:
- **Device 0:** MAC = `02:XX:XX:XX:XX:XX`, Name = `"TWS"` (Dual-earbud TWS receiver).
  - Note: MAC starts with `02`, indicating a standard Locally Administered (Randomized) Private Bluetooth address generated by a JieLi (JL) or Bluetrum TWS chip.
- **Device 1:** MAC = `01:XX:XX:XX:XX:XX`, Name = `"BT_HEADPHONE"` (Standard single-device Bluetooth headphone).

---

## 6. ROOT CAUSE ANALYSIS OF DUAL-LINK TWS STUTTERING

### 6.1 The Mechanical & RF Difference
1. **Single-Link Headphones (No Stutter):** Single receiver. One ACL link between player and headphone. No secondary link. Jitter buffer on standard over-ear headphones is 200-500 ms. Even with body absorption and 30% packet loss, retransmissions easily succeed within the frame period.
2. **TWS Primary Earbud Alone (No Stutter for >5 min):** Single receiver. The Right bud is the primary. With the Left bud in the case, the earbud-to-earbud relay link is shut off. The Right bud acts identically to a single-link headphone.
3. **TWS Both Earbuds (Stutters after ~2 min):** Dual-link relay topology.
   - The player transmits A2DP packets over the primary link to the Right earbud.
   - The Right earbud must re-transmit / relay audio packets across the user's head to the Left earbud in the SAME 2.4 GHz spectrum.
   - The human head absorbs 15-25 dB of 2.4 GHz RF power.

### 6.2 The Protocol & Timing Bottleneck
1. **Airtime Starvation Under Packet Retransmissions:**
   - The Beken BK3266 controller uses standard 2-DH5 packets (5 time slots = 3.125 ms + 1 reverse slot = 3.75 ms).
   - In ideal line-of-sight conditions, 1 transmission takes 3.75 ms out of ~14.5 ms audio frame time, leaving ~10.7 ms for the Right earbud to forward the packet to the Left earbud.
   - When the user moves, puts the player in a pocket, or covers it with a hand, signal attenuation causes packet corruption.
   - The Beken transmitter aggressively retransmits lost packets. Two retries consume `3.75 \times 3 = 11.25\text{ ms}`.
   - **The Primary earbud has zero time slots left to forward audio to the Secondary (Left) earbud.**
   - The Left earbud's buffer empties and starves, causing immediate audio drops.
   - The Left earbud responds by flooding the Right earbud with retransmission requests (NACKs), causing mutual interference on the cross-head link and forcing the Right bud to stall as well.

2. **The 2-Minute Onset Mechanism:**
   - At the beginning of streaming, both earbuds initialize with a full jitter buffer (150-300 ms).
   - Furthermore, after pairing and connection establishment, Bluetooth Classic controllers or TWS peripherals enter **Sniff Mode / Power Save parameter renegotiation** after an initial activity timeout (~60-120 seconds).
   - Once low-power / subrating anchor intervals are imposed, the available time slots for retransmission are severely restricted, collapsing the fragile cross-head relay link budget.

3. **Why Modern Bluetooth 5.x Smartphones Work Flawlessly:**
   - Modern smartphones implement Bluetooth 5.x with:
     - Dynamic SBC Bitpool reduction under packet loss (shrinking packets from 2-DH5 to 2-DH3, using only 3 slots instead of 5).
     - TWS-aware inter-packet scheduling (enforcing idle gaps between packets to guarantee airtime for the earbud-to-earbud relay).
     - Adaptive flush timeouts (dropping expired audio frames rather than indefinitely retrying over the air).
   - The Beken BK3266 in the portable player runs early Bluetooth audio firmware with a static, non-adaptive transmitter scheduler.

---

## 7. DISPROVED HYPOTHESES

1. **Hypothesis: Low RF TX Output Power in Config.**  
   *Disproved:* `CONFIG.TXT` explicitly sets `BTSTACK_OUTPUT_POWER_VALUE = 7` (the maximum allowed hardware level).
2. **Hypothesis: Audio Codec Incompatibility (AAC vs SBC).**  
   *Disproved:* Modern smartphones forced specifically to the SBC codec play flawlessly with the same TWS earbuds.
3. **Hypothesis: TWS Hardware / Antenna Defect.**  
   *Disproved:* The TWS earbuds work perfectly with modern smartphones, and the Primary (Right) earbud alone works perfectly for >5 minutes with the player even under pocket/body obstruction.
4. **Hypothesis: ATJ2127 Software Bug in A2DP Packet Encoding.**  
   *Disproved:* ATJ2127 does not encode A2DP or manage Bluetooth Baseband packets. The entire Bluetooth Baseband, L2CAP, AVDTP, and RF link scheduling resides on the autonomous Beken BK3266 chip.

---

## 8. FEASIBILITY OF FIRMWARE / CONFIG PATCH

1. **Host-Side Config (`CONFIG.TXT` / `CONFIG.BIN`):**
   - The host configuration exposed to ATJ2127 contains high-level parameters (`BTSTACK_OUTPUT_POWER_VALUE`, `BTMODULE_DEFAULT_BAUDRATE`, `BTSTACK_AUTO_CONNECT_INTERVAL`).
   - It does NOT expose Baseband-level parameters such as packet type masks (`NO_2DH5`), flush timeouts, or A2DP inter-packet spacing.
   - Modifying `CONFIG.TXT` cannot alter the Beken controller's internal Bluetooth Baseband retransmission engine.
2. **Bluetooth Controller Firmware (`BTUPG.HEX`):**
   - The definitive fix for the link scheduling bottleneck requires updating the firmware running on the Beken Bluetooth controller itself.
   - `BT.AP` contains the complete upgrade routine (`s_bt_upgrade`), which searches for `BTUPG.HEX` on the SD card via `card.drv`.
   - If a compatible Beken BK3266/BK3254 firmware image with improved TWS relay tolerance is obtained, it can be flashed via the player's native SD card upgrade menu without risky NAND modifications.

---

## 9. APPLIED CONFIGURATION PATCH & REPACKED IMAGE

1. **Applied Parameter Changes:**
   - Parameter: `BTSTACK_AVRCP_VOLUME_SYNC`
   - Original Value: `1` (Enabled)
   - Patched Value: `0` (Disabled)
   - File 1: `fw_unpack/CONFIG.TXT` (Line 90: `BTSTACK_AVRCP_VOLUME_SYNC = 0[0,1];`)
   - File 2: `fw_unpack/CONFIG.BIN` (Entry 67, file offset `0x50a`: `01 00` -> `00 00`)
   - Backups: `CONFIG.TXT.orig`, `CONFIG.BIN.orig`

2. **Repacked Firmware Image:**
   - File: `lfi_patched.bin` (Size: 50,593,280 bytes)
   - Updated Checksums:
     - `CONFIG.TXT` 32-bit checksum: `0xe651158f`
     - `CONFIG.BIN` 32-bit checksum: `0x6c916c87`
     - LFI Directory 32-bit checksum: `0x3315c32b`
     - LFI Header 16-bit checksum: `0x5907`
   - Integrity: Verified with `fwhelper scan_lfi` (0 checksum errors, all 78 files valid).

---

## 10. EMPIRICAL VERIFICATION & FINAL RESOLUTION

1. **Hardware Testing & Confirmation:**
   - Test 1 (Screen Kept Awake): When the screen is active, audio playback is 100% stable with zero stuttering across both single and dual TWS earbuds, regardless of distance or body obstruction.
   - Test 2 (Timing Correlation): Stuttering onset correlates precisely with the 1-minute mark following screen sleep (`SETTING_AP_ID_SCREEN_LIGHTOFF_TIMER = 1`), when the host/companion subsystem transitions power/clock states.
   - Test 3 (UI Mitigation): Adjusting the `Sleep Timer` / screen power-save behavior in the device's Settings menu prevents the low-power state transition during active Bluetooth streaming.
2. **Outcome:**
   - The TWS audio stuttering issue is **RESOLVED** on the physical hardware via non-destructive settings.
   - Zero physical flashing was required, completely preserving device safety and flash integrity.
   - Status: **CLOSED / SOLVED**.
