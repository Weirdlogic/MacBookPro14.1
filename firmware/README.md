# MacBook Pro 13" 2017 — Firmware extracted from macOS

This directory contains all proprietary firmware and calibration data
extracted from **macOS Ventura** running on the same MacBook Pro 13" 2017 (MacBookPro14,1).
These files are **not available in any Linux package** and must be taken from the macOS install.

---

## Directory layout

```
firmware/
├── bluetooth/                    ← Broadcom BCM4350C0 UART Bluetooth
│   ├── BCM4350C0.hcd             ← ready-to-use HCD for Linux (built by hex2hcd.py)
│   ├── hex2hcd.py                ← converter: macOS HEX → Linux HCD
│   └── source/
│       ├── BCM4350-MiniDriver-uart.hex   ← original macOS MiniDriver
│       └── BCM4350-Updater.hex           ← original macOS firmware patch
│
├── wifi/                         ← Broadcom BCM4350 WiFi
│   ├── brcmfmac4350-pcie.txt                          ← macOS NVRAM (generic fallback)
│   └── brcmfmac4350-pcie.Apple Inc.-MacBookPro14,1.txt ← macOS NVRAM (model-specific)
│
└── display/                      ← Built-in Retina LCD
    └── Color-LCD-MacBookPro14-1.icc  ← Apple factory color calibration (ICC profile)
```

---

## bluetooth/ — BCM4350C0 UART Bluetooth

### Hardware
| Key | Value |
|---|---|
| Chip | Broadcom BCM4350C0 |
| Transport | **UART** (ttyS4 / serial0) — not USB |
| macOS name | BCM\_4350 / BCM2E7C |
| Linux name | BCM4350C0 (die revision C0) |
| Firmware version | v134 c5628 (from macOS system\_profiler) |

### Not installed by default — read this before installing it by hand

The chip runs from internal ROM without any external firmware, and basic
Bluetooth (scan, pair, connect) works that way. **Neither `macbook_hardware_fixer.sh`
nor `bluetooth/bluetooth.sh` installs this patch.** The files are kept here as a
reference and for anyone who wants to experiment.

**No measured benefit.** On MacBookPro14,1 / Linux 7.1.8 / BlueZ 5.87, three
consecutive cold boots with nothing differing between arms but the file itself:

| | no patch | patch applied |
|---|---|---|
| Read Local Version | `08 FC 15 08 0F 00 86 61` | identical |
| Read Local Supported Features | `BF FE CF FE DB FF 7B 87` | identical |
| Read Buffer Size | `FD 03 C0 08 00 01 00` | identical |
| HCI revision | `0x15FC` | `0x15FC` |

No change in any field, and no bump in the HCI revision, which is the usual
signature of an applied Broadcom patchram. At a cold *unpatched* boot the chip
already reports `build 1532` rather than `0000`, so Apple's EFI most likely loads
an equivalent image at power-on and Linux would be rewriting it for nothing. The
choppy-A2DP reports this firmware was once credited with are just as consistent
with the baud-rate problem the SMC Reset addresses.

A warm `hci_uart` rebind is **not** a valid control here: this board reports
`No reset resource`, so unbind/bind never resets the chip and patch RAM survives.
Only a full power-off gives a clean arm.

**A real downside, measured.** Get the final `HCI_VS_Launch_RAM` address wrong by
four bytes and the chip accepts all 337 vendor commands and then stops answering
the UART completely:

```text
Bluetooth: hci0: BCM 'brcm/BCM.hcd' Patch
Bluetooth: hci0: command 0xfc18 tx timeout
Bluetooth: hci0: BCM: failed to write update baudrate (-110)
Bluetooth: hci0: BCM: Reset failed (-110)
```

No controller registered at all — strictly worse than having no patch file.
Reproduced on two independent A1708 boards. Recovering from it needs the file
removed **and** another full power-off, since patch RAM outlives a warm reboot.

Zero measured benefit against that failure mode is why this is opt-in.

If you can measure a difference on your machine, please open an issue with the
before/after output of `hcitool -i hci0 cmd 0x04 0x0001`.

The `linux-firmware` package does **not** ship BCM4350C0.hcd (it's an OEM
Apple file). It must be extracted from macOS.

### File origin
macOS ships the firmware as two Intel HEX files:

| File | Size | Purpose |
|---|---|---|
| `source/BCM4350-MiniDriver-uart.hex` | 20,513 bytes | Bootstrap loader (initialises UART comms) |
| `source/BCM4350-Updater.hex` | 181,600 bytes | Full firmware patch |

`hex2hcd.py` converts these to the `.hcd` format Linux expects:

```bash
# Rebuild BCM4350C0.hcd from source (Python 3, no dependencies):
python3 firmware/bluetooth/hex2hcd.py

# Verify output
od -A d -t x1 -N 8 firmware/bluetooth/BCM4350C0.hcd
# Expected: 4c fc ff 00 02 0d 00 ...  — opcode 0xFC4C (HCI_VS_Write_RAM), plen 0xFF,
# then the 4-byte little-endian address. There is no leading 01: that is the H4 UART
# packet-type byte, which the kernel transport prepends on the wire, so it must not
# be stored in the file.

tail -c 7 firmware/bluetooth/BCM4350C0.hcd | od -A n -t x1
# Expected: 4e fc 04 ff ff ff ff  — the final HCI_VS_Launch_RAM must use the
# default-entrypoint sentinel 0xFFFFFFFF. Launching at the updater's first segment
# address instead downloads cleanly and then leaves the controller mute on the UART.

sha256sum firmware/bluetooth/BCM4350C0.hcd
# f968320baf7109e19776d7b720a19f71babced1e675602df3632a40bdba6ab34
```

### Installing it by hand, if you want to try it anyway
Nothing in this repo does this for you. `brcm/BCM.hcd` is the only name the
kernel requests on this chip, so that is the only file worth placing:

```bash
sudo cp firmware/bluetooth/BCM4350C0.hcd /lib/firmware/brcm/BCM.hcd
```

Then **power off completely** and start again from the button. A warm reboot is
not enough to evaluate it, and — more importantly — is not enough to undo it if
it goes wrong. To go back:

```bash
sudo rm /lib/firmware/brcm/BCM.hcd
```

followed by another full power-off.

Verify:
```bash
journalctl -b -k | grep "hci0.*BCM"   # want: BCM: firmware 'brcm/BCM.hcd' Patch
                                      # and NO "Patch command N failed" lines
hciconfig hci0                         # should show: UP RUNNING, real BD Address

# "firmware Patch file not found, tried: brcm/BCM.hcd" means NO patch was applied.
# It is not cosmetic and it is not about a secondary file — the link above is missing.
```

### ⚠️ SMC Reset required once after migrating from macOS
macOS sets the chip baud rate to 3 Mbaud. Linux uses 115200 baud → chip times out →
`hci0` never appears. An SMC Reset clears the chip to factory default. Needed **once**:

1. Shutdown completely (not restart)
2. Hold for 10 s: **Shift(L) + Ctrl(L) + Option(L) + Power**
3. Release all keys, press Power normally

---

## wifi/ — BCM4350 WiFi NVRAM

### Hardware
| Key | Value |
|---|---|
| Chip | Broadcom BCM4350 |
| Bus | PCIe |
| Vendor:Device | 14e4:4350 |
| macOS platform | "hawaii" (C-4355\_\_s-C1) |
| Board ID | 0x170, boardrev 0x1177 |

### Why Linux benefits from this
WiFi already works out-of-the-box in Linux via the `brcmfmac` driver and the
firmware binary from `linux-firmware` (`brcmfmac4350-pcie.bin`).

However, the **NVRAM** (`.txt`) file encodes board-specific RF calibration:
TX power levels, antenna parameters, channel restrictions, 2.4/5 GHz tuning.
The macOS NVRAM is calibrated for this exact board (`boardid=0x170`).

Using the macOS NVRAM can improve:
- WiFi range (correct TX power limits)
- Stability at 5 GHz (tuned channel offsets)
- Regulatory compliance (correct country band limits)

### File origin
macOS stores NVRAM files at:
```
/usr/share/firmware/wifi/C-4355__s-C1/P-hawaii_M-YSBC_V-m__m-2.5.txt
```
Platform: `C-4355__s-C1` (BCM4355 silicon, revision C1 — shares firmware with BCM4350)
Board variant: `hawaii` (MacBook Pro 2016/2017 Wi-Fi board name)
Version: `2.5` (extracted from macOS Ventura; v2.5 has updated PA calibration vs v2.3:
improved `pa2ga0`/`pa2ga1` coefficients, `boardrev=0x1250`)

### Install on Ubuntu
`macbook_hardware_fixer.sh` handles this automatically (step 3). Manual install:

```bash
sudo cp firmware/wifi/brcmfmac4350-pcie.txt \
        /lib/firmware/brcm/brcmfmac4350-pcie.txt

# Model-specific name (kernel tries this first):
sudo cp "firmware/wifi/brcmfmac4350-pcie.Apple Inc.-MacBookPro14,1.txt" \
        "/lib/firmware/brcm/brcmfmac4350-pcie.Apple Inc.-MacBookPro14,1.txt"

# Reload driver:
sudo rmmod brcmfmac && sudo modprobe brcmfmac
```

Verify:
```bash
dmesg | grep brcmfmac   # should show: "brcmfmac: brcmf_fw_alloc_request: ...nvram"
```

---

## display/ — Apple factory LCD color calibration (ICC profile)

### Hardware
| Key | Value |
|---|---|
| Display | Built-in Retina LCD, 2560×1600 |
| Panel | LG LP133QD1 (13.3", IPS) |
| Profile type | ICC v2.16 display profile (mntr RGB XYZ) |
| CMM | Apple (appl) |
| White point | D65 (0.9496, 1.0000, 1.0890 XYZ) |

### Why Linux needs this
Ubuntu uses a generic sRGB color assumption for all displays. The MacBook Pro
panel has a **wider gamut than sRGB** (red/green primaries are outside sRGB).
Without the correct ICC profile:
- Reds and greens appear oversaturated
- White point is incorrect (warmer/cooler than calibrated)
- Any color-managed application (Firefox, Darktable, GIMP) renders wrong

### Profile content
| ICC tag | Content |
|---|---|
| `rXYZ` / `gXYZ` / `bXYZ` | Factory-measured RGB primaries in XYZ |
| `rTRC` / `gTRC` / `bTRC` | 1024-point tone response curves (per channel) |
| `wtpt` | D65 white point |
| `vcgt` | Video card gamma table (identity — TRC carries the calibration) |
| `vcgp` | Apple extended gamma parameters |

### File origin
macOS stores the per-display profile at:
```
/Library/ColorSync/Profiles/Displays/Color LCD-<UUID>.icc
```
The UUID (`C016EBBE-006D-7C90-E158-A8AFDA0A2266`) is unique per unit.
The profile included here was extracted from the specific MacBook Pro this
repo was built for. It should be correct for all MacBookPro14,1 units with
the same panel (LG LP133QD1).

### Install on Ubuntu
`macbook_hardware_fixer.sh` step 11 handles this automatically. Manual install:

```bash
sudo mkdir -p /usr/share/color/icc/macbook
sudo cp firmware/display/Color-LCD-MacBookPro14-1.icc \
        /usr/share/color/icc/macbook/

mkdir -p ~/.local/share/icc
cp firmware/display/Color-LCD-MacBookPro14-1.icc \
   ~/.local/share/icc/

# Assign via colormgr (replace DEVICE_ID with output of 'colormgr get-devices'):
colormgr import-profile firmware/display/Color-LCD-MacBookPro14-1.icc
colormgr device-add-profile <DEVICE_ID> \
    $(colormgr get-profiles | grep Color-LCD | awk '/Profile ID/{print $NF}')
```

Or use **GNOME Settings → Color** and select `Color-LCD-MacBookPro14-1` from the list.

---

## Re-extracting from macOS (if you need to redo this)

From a macOS terminal on the same MacBook Pro:

```bash
# Bluetooth firmware
cp /usr/share/firmware/bluetooth/BCM4350-MiniDriver-uart.hex firmware/bluetooth/source/
cp /usr/share/firmware/bluetooth/BCM4350-Updater.hex         firmware/bluetooth/source/
python3 firmware/bluetooth/hex2hcd.py

# WiFi NVRAM
cp "/usr/share/firmware/wifi/C-4355__s-C1/P-hawaii_M-YSBC_V-m__m-2.5.txt" \
   firmware/wifi/brcmfmac4350-pcie.txt
cp firmware/wifi/brcmfmac4350-pcie.txt \
   "firmware/wifi/brcmfmac4350-pcie.Apple Inc.-MacBookPro14,1.txt"

# Display ICC (UUID in filename varies per unit)
cp /Library/ColorSync/Profiles/Displays/Color\ LCD-*.icc \
   firmware/display/Color-LCD-MacBookPro14-1.icc
```
