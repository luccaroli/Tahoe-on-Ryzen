# Tahoe-on-Ryzen

macOS **Tahoe 26.7.2** running with OpenCore on a budget AMD PC: Ryzen 5 5500 + Radeon RX 6600 + Intel AX210 Wi-Fi. Full graphics acceleration (Metal 3), Wi-Fi, Bluetooth, Ethernet, and Xcode 26.5 building real iOS apps in the Simulator.

## Tested hardware

| Component | Model | Status |
|---|---|---|
| CPU | AMD Ryzen 5 5500 (6 cores / 12 threads) | ✅ AMD Vanilla patches (6 cores) |
| Motherboard | SOYO SY-KL A520M | ✅ |
| GPU | AMD Radeon RX 6600 (Navi 23) | ✅ Metal 3, via NootRX |
| Wi-Fi | Intel AX210 | ✅ itlwm + HeliPort |
| Ethernet | Realtek RTL8111 | ✅ RealtekRTL8111 |
| Audio | AMD HD Audio | ⚠️ AppleALC loaded, `alcid=1`, layout still being tuned |
| Bluetooth | Intel AX210 | ✅ IntelBluetoothFirmware + IntelBTPatcher + BlueToolFixup (tested on 26.7.2, Echo Dot paired as A2DP speaker) |
| AirDrop / Continuity | Intel AX210 | ⚠️ Limited (Handoff-level features only, no AirDrop/Hotspot) |
| RAM | 16 GB | ✅ |

## Versions

| Item | Version |
|---|---|
| OpenCore | 1.0.8 |
| macOS | Tahoe 26.7.2 (25G332) |
| SMBIOS | MacPro7,1 |
| Lilu | 1.7.2 |
| VirtualSMC | 1.3.8 |
| NootRX | 1.0.0 |
| AppleALC | 1.9.8 |
| itlwm | 2.3.0 (+ HeliPort) |
| IntelBluetoothFirmware | [2.5.2 (lshbluesky fork)](https://github.com/lshbluesky/IntelBluetoothFirmware/releases/tag/v2.5.2) |
| IntelBTPatcher | [2.5.2 (lshbluesky fork)](https://github.com/lshbluesky/IntelBluetoothFirmware/releases/tag/v2.5.2) |
| BlueToolFixup | [2.7.2 (BrcmPatchRAM)](https://github.com/acidanthera/BrcmPatchRAM/releases/tag/2.7.2) |
| RealtekRTL8111 | 3.0.4 |
| NVMeFix | 1.1.3 |
| RestrictEvents | 1.1.6 |
| AppleMCEReporterDisabler | 1.2 |

`AMDRyzenCPUPowerManagement` and `SMCAMDProcessor` are included but **disabled** in the config.

## ⚠️ Before you use it: generate your own SMBIOS

`config.plist` ships with `SystemSerialNumber`, `MLB` and `SystemUUID` set to `CHANGE_ME` and `ROM` zeroed. Generate your own with [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) (model **MacPro7,1**), put them in `PlatformInfo > Generic`, and use your network card's MAC address as `ROM`. Check on [Apple's coverage site](https://checkcoverage.apple.com) that the serial comes back **invalid** before signing in with your Apple ID.

## Key settings

- **GPU**: the native RX 6600 driver kernel panicked (PPLIB) in WindowServer. **NootRX** fixes it.
- **Wi-Fi on Tahoe**: itlwm 2.3.0 only works with `Kernel > Quirks > DisableIoMapper = true`.
- **Wi-Fi**: use **itlwm + HeliPort** instead of AirportItlwm, which breaks iMessage, AirDrop and Hotspot.
- **Bluetooth on Tahoe**: the official OpenIntelWireless IntelBTPatcher stops at Sequoia, so this EFI uses the Tahoe-ready [lshbluesky fork](https://github.com/lshbluesky/IntelBluetoothFirmware) (v2.5.2) plus [BlueToolFixup 2.7.2](https://github.com/acidanthera/BrcmPatchRAM) from Acidanthera (macOS 26 `bluetoothd` patches). Kexts load right after itlwm in this order: IntelBluetoothFirmware, IntelBTPatcher, BlueToolFixup. IntelBluetoothInjector is **not** used. NVRAM adds `bluetoothExternalDongleFailed = 00` and `bluetoothInternalControllerInfo = 14 zero bytes` (also listed under `NVRAM > Delete`). Do a **Reset NVRAM** from the OpenCore menu after applying.
- **Bluetooth USB port**: the AX210 Bluetooth (`8087:0032`) shows up on the internal USB 2.0 port `PO10` and works **without a USBMap** (connector type reported as 0, not 255). If your Bluetooth stays Off or shows a fake `BCM_4350C2` chipset, make a USB map (e.g. USBToolBox) and set that port to Internal (255).
- `SecureBootModel = Disabled`.
- boot-args: `alcid=1 keepsyms=1 debug=0x100`.
- **AMD Vanilla** kernel patches are set for **6 cores**. If your Ryzen has a different core count, adjust the 3 `algrey - Force cpuid_cores_per_package` patches following [AMD Vanilla](https://github.com/AMD-OSX/AMD_Vanilla).

## BIOS

- CSM: **off** (pure UEFI boot)
- SVM: on
- Above 4G Decoding: **on**
- Resizable BAR: **off**
- Secure Boot: off

## Installation (summary)

1. Download the recovery with `macrecovery.py` from OpenCorePkg and put it on a FAT32 USB drive along with this `EFI` folder.
2. Disconnect your other drives during installation.
3. Tip: the **Sonoma** recovery was the most stable on this hardware. Install Sonoma, then upgrade to Tahoe through Software Update.
4. After installing, copy the `EFI` folder to the EFI partition of the macOS drive.

## Dual boot with Ubuntu (GRUB)

To boot macOS straight from GRUB without going through two menus, add this to `/etc/grub.d/40_custom`:

```
menuentry "macOS" --class macosx {
    insmod part_gpt
    insmod fat
    search --no-floppy --fs-uuid --set=root YOUR-MAC-EFI-UUID
    chainloader /EFI/BOOT/BOOTx64.efi
}
```

In `config.plist`, set `Misc > Boot > ShowPicker = false` (hold Esc to show the OpenCore menu). Then run `sudo update-grub`.

## iOS development

- **Xcode 26.5 (Universal)** works. **Xcode 27+ is Apple Silicon only** and won't launch on a Hackintosh ("bad CPU type").
- The iOS 26.5 Simulator works, but it's slow on a spinning hard drive. Use an SSD.
- Homebrew refuses to install on Intel/AMD hardware on Tahoe. Install Node, gh, Ruby, etc. manually under `~/.local`.

## Limitations

- No virtualization inside macOS on AMD: no Docker, no Android emulator.
- AirDrop, Handoff and Continuity are limited with Intel Wi-Fi.
- USB map not done yet.

## Credits

[Acidanthera](https://github.com/acidanthera), [AMD-OSX](https://github.com/AMD-OSX), [ChefKiss (NootRX)](https://github.com/ChefKissInc/NootRX), [OpenIntelWireless](https://github.com/OpenIntelWireless), [lshbluesky (Tahoe Intel Bluetooth fork)](https://github.com/lshbluesky/IntelBluetoothFirmware), [Mieze (RealtekRTL8111)](https://github.com/Mieze/RTL8111_driver_for_OS_X) and the [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/).

If this helped you, leave a ⭐! Use at your own risk.

## Using an AI agent?

This repo includes [`AGENTS.md`](AGENTS.md) with step-by-step instructions any AI coding agent (Claude Code, Codex, Grok, Cursor, Copilot) can follow to adapt this EFI to your hardware safely. Just clone the repo and ask your agent: *"Read AGENTS.md and help me set this up for my PC."*
