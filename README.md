# actions_flash

[![Language](https://img.shields.io/badge/language-C99%20%2F%20ASM-blue.svg)](#)
[![Targets](https://img.shields.io/badge/SoC-ATJ2127%20(MIPS)%20%7C%20ATJ2157%20(ARM)-orange.svg)](#)
[![USB](https://img.shields.io/badge/USB-libusb--1.0%20%7C%20ttyUSB-success.svg)](#)
[![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20macOS%20%7C%20Windows-lightgrey.svg)](#)

A low-level **ADFU (Actions Device Firmware Upgrade)** protocol client, bare-metal payload execution framework, raw NAND flash dumper, **LFI (Logical Firmware Image)** recovery/unpacking suite, and hardware reverse-engineering reference for portable media players powered by **Actions Semiconductor ATJ2127 (MIPS32/MIPS16)** and **ATJ2157 (ARM)** SoCs.

---

## Key Features

- **Mode Switching & Device Probing**: Switch devices directly from normal USB Mass Storage mode (`10d6:1101`) into ADFU recovery mode (`10d6:10d6`) via vendor SCSI commands (`adfu_reboot`) without needing physical boot-key combinations.
- **Arbitrary Memory Access & Code Execution**: Read SRAM/ROM, upload custom binaries into on-chip memory, and execute bare-metal code over USB (`read_mem`, `read_mem2`, `write_mem`, `exec_ret`, `simple_exec`).
- **Custom Bare-Metal Payloads (MIPS & ARM)**:
  - `adfus`: Clean, reverse-engineered replacement for the vendor ADFU server stub, unlocking extended memory reads, system resets, and large block transfers.
  - `nandhwsc`: Directly queries the on-chip NAND controller to identify flash chip geometry and vendor IDs.
  - `nandread`: Raw NAND flash page/sector reader capable of dumping boot records (`MBREC`, `BREC`) and scanning non-contiguous NAND blocks to reconstruct firmware images (`find_lfi`).
- **Firmware Repair & Unpacking (`fwhelper`)**:
  - `lfi_repair`: Automatically cross-references primary and backup LFI copies dumped from NAND flash, solving page-level checksums to repair bit-rot and damaged NAND pages.
  - `scan_lfi` / `unpack_lfi`: Validates LFI header, directory, and per-file checksums, inspects `yqhx` executable headers, and extracts all embedded firmware modules (`.AP`, `.DRV`, `.AL`, `.STY`, `CONFIG.BIN`, etc.).
- **Reverse-Engineering Documentation ([RE_NOTES.md](RE_NOTES.md))**: Detailed architectural notes covering the Actions `YQHX` 1.5 KB overlay banking model, MIPS16 `break 0` syscall ABI, ATJ2127 hardware registers (DMA, UART, GPIO, Watchdog), and the companion Beken BK3266 Bluetooth subsystem protocol.

---

## Repository Structure

| Path | Description |
| :--- | :--- |
| [`actions_dump.c`](actions_dump.c) | Main host CLI utility communicating with the target over `libusb-1.0` or USB serial (`USBC`/`USBS` CBW/CSW protocol). |
| [`Makefile`](Makefile) | Build configuration for `actions_dump` (`LIBUSB=1` by default, or `LIBUSB=0` for Linux `ttyUSB` serial). |
| [`fwhelper/`](fwhelper/) | Host utility (`fwhelper`) for inspecting, repairing (`lfi_repair`), and unpacking (`unpack_lfi`) LFI firmware images and `yqhx` binaries. |
| [`payload/`](payload/) | Bare-metal **MIPS32/MIPS16** payloads (`adfus`, `nandread`, `nandhwsc`, `hello`) targeting the **ATJ2127** SoC. |
| [`payload_arm/`](payload_arm/) | Bare-metal **ARM** payloads (`adfus`, `nandread`, `nandhwsc`, `hello`) targeting the **ATJ2157** SoC. |
| [`RE_NOTES.md`](RE_NOTES.md) | Deep-dive reverse-engineering specification of the ATJ2127 architecture, `YQHX` overlay loader, syscall ABI, hardware registers, and Bluetooth UART protocol. |

---

## Hardware & USB Overview

Actions ATJ2127/ATJ2157 players expose two primary USB identities:

| Mode | USB VID:PID | Interface Description | How to Enter |
| :--- | :--- | :--- | :--- |
| **Player / Flash Disk Mode** | `10d6:1101` | `HS USB FlashDisk` (Mass Storage) | Connect device normally while powered on. |
| **ADFU Recovery Mode** | `10d6:10d6` | `Actions USB 2.0 (HS) WinUSB Device` | Run `./actions_dump --id 10d6:1101 adfu_reboot`, **or** power off and hold the device's designated boot key while connecting USB. |

> [!WARNING]
> **Hardware Safety & Power Switch Behavior**
> - **Software Power Switches**: Many players have a power switch that looks physical but is actually wired to a GPIO (noticeable by a ~0.5s delay when turning off). Such devices remain powered in ADFU mode even after unplugging USB. Holding the boot key for several seconds or issuing the `reset` command after loading `adfus.bin` helps exit ADFU mode. If a payload hangs and the device lacks a physical battery disconnect, you may need to wait for the battery to drain.
> - **Flash Writes**: Commands such as `write_flash` and `read_lfi` rely on vendor `fwscfNNN.bin` flash drivers and can permanently brick the device if mismatched binaries or addresses are used. Prefer read-only extraction (`read_brec`, `find_lfi`, `read_nand`) via `nandread.bin`.

---

## Building from Source

### 1. Building `actions_dump` and `fwhelper`

#### Option A: Using `libusb-1.0` (Recommended — Linux, macOS, Windows/MSYS2)

Ensure `libusb-1.0` development headers are installed:
- **Debian / Ubuntu**: `sudo apt install build-essential libusb-1.0-0-dev`
- **macOS (Homebrew)**: `brew install libusb`
- **Windows (MSYS2)**: Install `mingw-w64-x86_64-libusb` and configure the WinUSB driver via [Zadig](https://github.com/libusb/libusb/wiki/Windows#driver-installation).

Build both host tools:
```bash
# Build actions_dump
make

# Build fwhelper
make -C fwhelper
```

*(On macOS with Homebrew on Apple Silicon, if `/opt/homebrew/include` is not in your default compiler path, run `make CFLAGS="-O2 -Wall -DUSE_LIBUSB=1 -I/opt/homebrew/include" LIBS="-L/opt/homebrew/lib -lusb-1.0"`).*

#### Option B: Using Linux Generic USB Serial (`LIBUSB=0`, Linux Only)

```bash
make LIBUSB=0
```

When using serial mode, bind the kernel generic USB-serial driver before running `actions_dump` (required once per boot):
```bash
sudo modprobe ftdi_sio
echo 10d6 10d6 | sudo tee /sys/bus/usb-serial/drivers/generic/new_id
```

### 2. Building Target Payloads (`payload/` & `payload_arm/`)

To compile the bare-metal payloads from source, a cross-compiler toolchain (`mipsel` for ATJ2127 in [`payload/`](payload/README.md) or `arm-none-eabi` for ATJ2157 in [`payload_arm/`](payload_arm/README.md)) is required:

```bash
# Example: Build ATJ2127 (MIPS) payloads (adfus.bin, nandread.bin, nandhwsc.bin, hello.bin)
make -C payload all

# Example: Build ATJ2157 (ARM) payloads
make -C payload_arm all
```

---

## Step-by-Step Usage Workflow

### 1. Reboot from Mass Storage Mode into ADFU Mode

If your player is connected normally as a USB drive (`10d6:1101`), switch it into ADFU mode (`10d6:10d6`):

```bash
sudo ./actions_dump --id 10d6:1101 adfu_reboot
```

Query the Boot ROM chip identifier:
```bash
sudo ./actions_dump adfu_info
# ATJ2127 returns "\0CADFUD\x10\xd6..." -> guess: chip = ATJ2127
# ATJ2157 returns "\0CADFUD\x30\x51..." -> guess: chip = ATJ2157
```

### 2. Dump SRAM and Boot ROM

Using the Boot ROM's built-in `read_mem` command (note: the first 96 KB covering the Boot ROM reads as zeroes via Boot ROM `read_mem`):
```bash
sudo ./actions_dump read_mem 0xbfc00000 256K ram_dump.bin
```

To dump the full 256 KB including the **Boot ROM** (mapped at `0x9fc00000`), first load `adfus.bin` (`0xbfc18000` for ATJ2127, `0x118000` for ATJ2157) and use `read_mem2`:
```bash
# ATJ2127 (MIPS)
sudo ./actions_dump \
  simple_switch 0xbfc18000 payload/adfus.bin \
  read_mem2 0x9fc00000 256K rom_and_ram_dump.bin

# ATJ2157 (ARM)
sudo ./actions_dump \
  chip 2157 \
  simple_switch 0x118000 payload_arm/adfus.bin \
  read_mem2 0x00100000 256K rom_and_ram_dump.bin
```

### 3. Dump Boot Records (`MBREC`, `BREC`) & Full Firmware (`LFI`)

Once `adfus.bin` is active in RAM, use `payload/nandread.bin` to extract the Master Boot Record (`mbrec`), Boot Record (`brec`), and the Logical Firmware Image (`lfi`) directly from NAND flash:

```bash
# 1. Switch to adfus.bin (only once per ADFU session)
sudo ./actions_dump simple_switch 0xbfc18000 payload/adfus.bin

# 2. Dump MBREC and BREC (copy 0 and backup copy 1)
sudo ./actions_dump read_brec payload/nandread.bin mbrec0.bin 0 brec0.bin
sudo ./actions_dump read_brec payload/nandread.bin mbrec1.bin 1 brec1.bin

# 3. Locate and dump the LFI firmware chain (outputs primary + backup copies concatenated)
sudo ./actions_dump find_lfi payload/nandread.bin 0 lfi_raw.bin

# 4. Reboot the player back to normal mode
sudo ./actions_dump reset
```

### 4. Repair & Unpack the LFI Firmware Image (`fwhelper`)

Raw NAND dumps acquired via `find_lfi` contain two copies of the firmware back-to-back and may have isolated bit flips or bad page reads. Use `fwhelper` to repair damaged pages against the backup copy and extract all system files:

```bash
# 1. Repair damaged pages using the dual-copy LFI dump
#    Syntax: ./fwhelper/fwhelper <raw_dump> lfi_repair <fw_sectors> <pages_per_block> <page_size> <repaired_out>
./fwhelper/fwhelper lfi_raw.bin lfi_repair 0x18200 128 2048 lfi_repaired.bin

# 2. Verify LFI header, directory, and file checksums
./fwhelper/fwhelper lfi_repaired.bin scan_lfi

# 3. Extract all firmware files (.AP, .DRV, .AL, CONFIG.BIN, CONFIG.TXT, etc.)
mkdir -p fw_unpack && cd fw_unpack
../fwhelper/fwhelper ../lfi_repaired.bin unpack_lfi
```

---

## Command Reference

### `actions_dump` Global Options

| Option | Description |
| :--- | :--- |
| `--id <VID:PID>` | Override target USB Vendor/Product ID in hex (default: `10d6:10d6`). Use `10d6:1101` for Flash Disk mode. |
| `--tty <path>` | Serial device path when compiled with `LIBUSB=0` (default: `/dev/ttyUSB0`). |
| `--wait <sec>` | Seconds to wait/poll for USB device connection before timing out (default: `300`). |
| `--verbose <lvl>` | Verbosity level (`0` = normal, `1+` = dump USB packets). |

### `actions_dump` Commands

Commands can be chained sequentially in a single invocation.

| Category | Command | Description |
| :--- | :--- | :--- |
| **Configuration** | `chip <2127\|2157>` | Select target SoC architecture (`2127` default, `2157` for ARM). |
| | `blk_size <size>` | Set USB transfer block size (`1` to `0x4000`). |
| | `timeout <ms>` | Set USB transfer timeout in milliseconds. |
| | `verbose <lvl>` | Change runtime logging verbosity. |
| **Flash Disk Mode** | `inquiry` | Print standard SCSI device inquiry info (`10d6:1101`). |
| | `adfu_reboot` | Send vendor SCSI sequence (`0xCC` / `0xCB`) to reboot into ADFU mode (`10d6:10d6`). |
| **Boot ROM** | `adfu_info` | Query Boot ROM identification string (`\0CADFUD...`). |
| | `write_mem <addr> <off> <size> <file>` | Upload `<file>` into device memory at `<addr>` (`size = 0` means until EOF). |
| | `read_mem <addr> <size> <file>` | Read memory range to `<file>` (cannot read Boot ROM region). |
| | `switch <addr>` | Jump and switch USB command handler to `adfus` loaded at `<addr>`. |
| | `simple_switch <addr> <file>` | Shorthand for `write_mem <addr> 0 0 <file> switch <addr>`. |
| | `exec_ret <addr> <ret_size>` | Execute code at `<addr>` and read `<ret_size>` bytes of response (`-1` for variable length). |
| | `simple_exec <addr> <file> <ret_size>` | Shorthand for `write_mem <addr & ~1> 0 0 <file> exec_ret <addr> <ret_size>`. |
| **Extended (`adfus`)** | `reset` | Trigger hardware watchdog reset to reboot the device into normal player mode. |
| | `read_mem2 <addr> <size> <file>` | Read any memory region (including Boot ROM) using a tiny copy stub at `0xbfc1e000` / `0x11e000`. |
| | `read_brec <nandread.bin> <mbrec.bin> <idx> <brec.bin>` | Dump Master Boot Record (`mbrec.bin`) and Boot Record (`brec.bin`, index `0` or `1`). |
| | `read_nand <nandread.bin> <row> <count> <file>` | Read `<count>` raw NAND pages starting at `<row>`. |
| | `find_lfi <nandread.bin> <brec_idx> <lfi.bin>` | Scan NAND tags using BREC metadata and dump primary + backup LFI firmware copies. |
| | `read_lfi <addr> <size> <file>` | Read logical firmware sectors (requires vendor `fwscfNNN.bin` initialized). |
| | `write_flash <sector> <off> <size> <file>` | Write data to flash memory (requires vendor `fwscfNNN.bin` initialized). |

### `fwhelper` Commands

| Command | Description |
| :--- | :--- |
| `./fwhelper <lfi.bin> scan_lfi` | Validate LFI header/directory checksums and list all contained files with `yqhx` metadata. |
| `./fwhelper <lfi.bin> unpack_lfi` | Validate and extract all files from `<lfi.bin>` into the current working directory. |
| `./fwhelper <MODULE.AP> scan_file` | Parse and print section offsets, load addresses, and entry points of a `yqhx` binary. |
| `./fwhelper <raw.bin> lfi_repair <sectors> <pages_per_blk> <page_size> <out.bin>` | Reconstruct a clean LFI image from a dual-copy dump by repairing corrupted NAND pages. |

---

## Repeating the Factory Flashing Sequence (ATJ2127 Reference)

When captured via `usbmon` / Wireshark, the official vendor firmware updater executes the following sequence using binaries extracted from a vendor `.fw` package:

```bash
sudo ./actions_dump \
  simple_switch 0xbfc18000 adfus.bin \
  write_mem 0xbfc1e000 0 0 nandhwsc.bin exec_ret 0xbfc1e000 0x9c \
  write_mem 0x9fc24c00 0 0 nandinfo.bin \
  timeout 30000 \
  write_mem 0xbfc1e000 0 0 fwsc.bin exec_ret 0xbfc1e200 512 \
  write_flash 0x40000000 0 0 mbrec.bin \
  write_flash 0x46000000 0 0 brec.bin \
  write_flash 0xc0000000 0 0 fwim.bin \
  write_mem 0x9fc25400 0 0 flash_end.bin \
  write_flash 0xff000000 0 0 flash_end.bin \
  reset
```

---

## Running Without `sudo` on Linux (`udev` Rules)

Create `/etc/udev/rules.d/80-actions.rules` with the following configuration:

```udev
# Actions Semiconductor ADFU & Flash Disk modes
SUBSYSTEMS=="usb", ATTRS{idVendor}=="10d6", ATTRS{idProduct}=="10d6", MODE="0666", TAG+="uaccess"
SUBSYSTEMS=="usb", ATTRS{idVendor}=="10d6", ATTRS{idProduct}=="1101", MODE="0666", TAG+="uaccess"
```

Reload `udev` rules:
```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

---

## Reverse Engineering Notes

See **[RE_NOTES.md](RE_NOTES.md)** for an in-depth technical reference on the ATJ2127 platform, including:
1. **Dual-Chip Architecture**: Host ATJ2127 (MIPS32/MIPS16) + Companion Beken BK3266/BK3254 Bluetooth Audio SoC over 1.5 MBaud DMA UART.
2. **`YQHX` Binary Format & Overlay Banking**: How 11+ code banks are dynamically swapped into a 1.5 KB RAM execution window (`0xbfc35a00`–`0xbfc36000`) using tagged function pointers (`0x73XX5aXX`).
3. **MIPS16 System Call ABI**: Breakpoint exception (`li v1, <id>; break 0`) dispatch tables and kernel multiplexer (`Import 15`) selectors.
4. **Hardware Register Map**: Memory-mapped I/O addresses (`0xc0000000`–`0xc012001c`) for clocks, GPIO, pin muxing, DMA channels, UART, and watchdog timers.
5. **Bluetooth UART Framing & A2DP TWS Analysis**: Packet structure, checksum algorithm, command opcodes, and RF/airtime root-cause analysis for dual-earbud TWS relay links.

---

## Useful Links & References

- [`atjboottool` (Rockbox)](https://github.com/Rockbox/rockbox/tree/master/utils/atj2137/atjboottool) — Tool for decrypting Actions `.fw` firmware update containers.
- [`atj2127decrypt`](https://github.com/nfd/atj2127decrypt) — Actions ATJ2127 firmware decryption utility.
- [`PD196_ATJ2127`](https://github.com/Suber/PD196_ATJ2127) — Hardware documentation and resources for ATJ2127 devices.

---

## License

This software is provided **"AS IS"**, without warranty of any kind, express or implied. Use at your own risk.
