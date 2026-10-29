# AGENTS.md

Instructions for AI coding agents (Claude, Codex, Grok, Cursor, Copilot, etc.) helping a user adapt this OpenCore EFI to their own PC and install macOS Tahoe.

Read `README.md` first. It has the tested hardware, versions, BIOS settings and known issues.

## Safety rules (non-negotiable)

1. **Never format, partition, or write to any disk without an explicit "yes" from the user** after you have shown them the exact disk model and size (`lsblk -o NAME,MODEL,SIZE` on Linux, `diskutil list` + `diskutil info diskN` on macOS). Disk names like `sda` or `disk2` change between boots and OSes; always confirm by model and size.
2. Recommend the user **disconnects every other drive** during installation.
3. **Always back up the existing EFI** before replacing it.
4. **BIOS changes are made by the user.** Guide them step by step; you can't touch firmware settings.
5. **Never commit or share serials.** `SystemSerialNumber`, `MLB`, `SystemUUID` and `ROM` are unique per machine. Never ask the user to paste passwords, sudo passwords, or Apple ID credentials into chat; have them type passwords only in the OS prompt.
6. **Don't invent versions or compatibility.** Check the GitHub releases of each project before claiming support.

## Step 1: Identify the hardware

On Linux (live USB is fine):

```bash
lscpu | grep -E "Model name|^CPU\(s\)|Core\(s\) per socket"
lspci -nn | grep -Ei "vga|3d|network|ethernet|audio|sata|nvme|usb"
sudo dmidecode -t baseboard | grep -E "Manufacturer|Product"
lsblk -o NAME,MODEL,SIZE,TYPE
```

On Windows: `wmic cpu get name,NumberOfCores`, Device Manager > Properties > Details > Hardware Ids.

Compare against the table in `README.md`. Then adapt:

| Hardware | What to change |
|---|---|
| **CPU core count** | Edit the 3 `algrey - Force cpuid_cores_per_package` patches in `Kernel > Patch` (`Replace` value) per [AMD_Vanilla](https://github.com/AMD-OSX/AMD_Vanilla). This EFI is set for **6 cores**. |
| **Intel CPU** | This EFI is AMD-only. Start from the Dortania guide for your Intel generation instead. |
| **GPU** | RX 6600/6600 XT/6650 XT (Navi 23) need **NootRX**. Other supported AMD GPUs (RX 5000, RX 6800/6900) usually need no GPU kext but may need `agdpmod=pikera`. NVIDIA (Kepler+), Intel Arc and RX 7000 are **not supported** on Tahoe. |
| **Wi-Fi** | Intel: keep `itlwm.kext` + HeliPort and `DisableIoMapper = true`. Broadcom/Fenvi: remove itlwm, use native or AirportBrcmFixup. |
| **Ethernet** | Realtek RTL8111: keep `RealtekRTL8111.kext`. Intel I225/I226: `AppleIGC.kext`. Intel I211: `SmallTreeIntel82576.kext`. Remove kexts you don't need. |
| **Audio** | Find the codec, then pick a layout-id from the [AppleALC supported codecs list](https://github.com/acidanthera/AppleALC/wiki/Supported-codecs) and set `alcid=N` in boot-args. |

Every kext added or removed in `EFI/OC/Kexts` must also be added or removed in `Kernel > Add` in `config.plist`, in load order (Lilu first).

## Step 2: Generate SMBIOS

```bash
git clone https://github.com/corpnewt/GenSMBIOS && cd GenSMBIOS && python3 GenSMBIOS.py
```

Pick option "Generate SMBIOS", model `MacPro7,1`. Put the values in `PlatformInfo > Generic`:
`SystemSerialNumber`, `MLB`, `SystemUUID`. Set `ROM` to the 6-byte MAC address of the main NIC. The serial must show as **invalid** at https://checkcoverage.apple.com.

## Step 3: Validate the config

```bash
plutil -lint EFI/OC/config.plist      # macOS
# or, any OS: ocvalidate from the matching OpenCorePkg release (1.0.8)
./ocvalidate EFI/OC/config.plist
```

`ocvalidate` must come from the **same OpenCore version** as `EFI/OC/OpenCore.efi` (1.0.8 here).

## Step 4: BIOS (user does this)

CSM off, Secure Boot off, SVM on, Above 4G Decoding on, Resizable BAR off. On some boards: disable Fast Boot.

## Step 5: Make the installer USB

```bash
git clone --depth 1 https://github.com/acidanthera/OpenCorePkg
cd OpenCorePkg/Utilities/macrecovery
# Sonoma 14 recovery (most reliable on this hardware); upgrade to Tahoe later via Software Update
python3 macrecovery.py -b Mac-827FAC58A8FDFA22 -m 00000000000000000 download
```

Format the USB as **FAT32, GPT** (confirm the device with the user first), then copy `com.apple.recovery.boot/` and the adapted `EFI/` to its root.

## Step 6: Install and post-install

1. Boot the USB, pick the recovery, erase the target disk as APFS (GUID), install.
2. After first boot, mount the EFI partition of the macOS disk (`sudo diskutil mount diskNs1`), back up anything there, and copy `EFI/` to it.
3. Install HeliPort (Intel Wi-Fi) and add it to Login Items.
4. Turn off automatic macOS updates before checking that kexts support the next version.

## Verify it worked

```bash
sw_vers
kextstat | grep -E "Lilu|NootRX|itlwm|AppleALC|RealtekRTL8111"
system_profiler SPDisplaysDataType | grep -E "Chipset|Metal"
```

Expected: the macOS version, the kexts loaded, and `Metal Support: Metal 3`.

## Troubleshooting map

| Symptom | Likely fix |
|---|---|
| Stuck at "Waiting for root device" / no installer | Check `SecureBootModel = Disabled`, USB kexts/map, try a different port |
| Panic mentioning `PPLIB` / WindowServer black screen on Navi 23 | NootRX missing or disabled |
| HeliPort says "itlwm not running" on Tahoe | Set `DisableIoMapper = true` |
| Sequoia recovery hangs | Install Sonoma, then upgrade |
| Xcode says "bad CPU type" | Xcode 27+ is Apple Silicon only, use Xcode 26.x Universal |
| `invalid parameter` at OpenCore picker with `ShowPicker = false` | Show the picker (hold Esc), select macOS, press Ctrl+Enter to set it as default |

## Useful references

- [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/)
- [AMD_Vanilla patches](https://github.com/AMD-OSX/AMD_Vanilla)
- [OpenCorePkg releases](https://github.com/acidanthera/OpenCorePkg/releases)
- [NootRX](https://github.com/ChefKissInc/NootRX)
- [itlwm / HeliPort](https://openintelwireless.github.io/)
