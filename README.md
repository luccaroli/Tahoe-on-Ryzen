# Tahoe-on-Ryzen

macOS **Tahoe 26.7.2** rodando com OpenCore em um PC AMD de baixo custo: Ryzen 5 5500 + Radeon RX 6600 + Wi-Fi Intel AX210. Com aceleração gráfica (Metal 3), Wi-Fi, Ethernet e Xcode 26.5 compilando apps iOS de verdade no Simulador.

> 🇺🇸 OpenCore EFI for macOS Tahoe on a Ryzen 5 5500 / A520M / RX 6600 / Intel AX210 build. Tested and working, including Xcode 26.5 and the iOS 26.5 Simulator. English summary at the bottom.

## Hardware testado

| Componente | Modelo | Status |
|---|---|---|
| CPU | AMD Ryzen 5 5500 (6 núcleos / 12 threads) | ✅ AMD Vanilla patches (6 núcleos) |
| Placa-mãe | SOYO SY-KL A520M | ✅ |
| GPU | AMD Radeon RX 6600 (Navi 23) | ✅ Metal 3, via NootRX |
| Wi-Fi | Intel AX210 | ✅ itlwm + HeliPort |
| Ethernet | Realtek RTL8111 | ✅ RealtekRTL8111 |
| Áudio | AMD HD Audio | ⚠️ AppleALC carregado, `alcid=1`, layout ainda em ajuste |
| Bluetooth | Intel AX210 | ❌ Ainda não configurado |
| RAM | 16 GB | ✅ |

## Versões

| Item | Versão |
|---|---|
| OpenCore | 1.0.8 |
| macOS | Tahoe 26.7.2 (25G332) |
| SMBIOS | MacPro7,1 |
| Lilu | 1.7.2 |
| VirtualSMC | 1.3.8 |
| NootRX | 1.0.0 |
| AppleALC | 1.9.8 |
| itlwm | 2.3.0 (+ HeliPort) |
| RealtekRTL8111 | 3.0.4 |
| NVMeFix | 1.1.3 |
| RestrictEvents | 1.1.6 |
| AppleMCEReporterDisabler | 1.2 |

`AMDRyzenCPUPowerManagement` e `SMCAMDProcessor` estão incluídos, mas **desativados** no config.

## ⚠️ Antes de usar: gere seu próprio SMBIOS

O `config.plist` vem com `SystemSerialNumber`, `MLB` e `SystemUUID` como `CHANGE_ME` e o `ROM` zerado. Gere os seus com o [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) (modelo **MacPro7,1**), coloque em `PlatformInfo > Generic` e use o endereço MAC da sua placa de rede como `ROM`. Confira no [site de cobertura da Apple](https://checkcoverage.apple.com) que o serial dá **inválido** antes de entrar com o Apple ID.

## Configurações importantes

- **GPU**: o driver nativo da RX 6600 dava kernel panic (PPLIB) no WindowServer. O **NootRX** resolve.
- **Wi-Fi no Tahoe**: o itlwm 2.3.0 só funciona com `Kernel > Quirks > DisableIoMapper = true`.
- **Wi-Fi**: use **itlwm + HeliPort** em vez de AirportItlwm, porque o AirportItlwm atrapalha iMessage, AirDrop e Hotspot.
- `SecureBootModel = Disabled`.
- boot-args: `alcid=1 keepsyms=1 debug=0x100`.
- Patches de kernel **AMD Vanilla** já configurados para **6 núcleos**. Se o seu Ryzen tiver outra quantidade de núcleos, ajuste os 3 patches `algrey - Force cpuid_cores_per_package` conforme o [AMD Vanilla](https://github.com/AMD-OSX/AMD_Vanilla).

## BIOS

- CSM: **desligado** (boot UEFI puro)
- SVM: ligado
- Above 4G Decoding: **ligado**
- Resizable BAR: **desligado**
- Secure Boot: desligado

## Instalação (resumo)

1. Baixe o recovery com o `macrecovery.py` do OpenCorePkg e grave num pendrive FAT32 junto com esta pasta `EFI`.
2. Desconecte os outros discos durante a instalação.
3. Dica: o recovery do **Sonoma** foi o mais estável neste hardware. Instale o Sonoma e depois atualize para o Tahoe pela Atualização de Software.
4. Depois de instalar, copie a `EFI` para a partição EFI do disco do macOS.

## Dual boot com Ubuntu (GRUB)

Para abrir o macOS pelo GRUB sem passar por dois menus, adicione em `/etc/grub.d/40_custom`:

```
menuentry "macOS" --class macosx {
    insmod part_gpt
    insmod fat
    search --no-floppy --fs-uuid --set=root UUID-DA-EFI-DO-MAC
    chainloader /EFI/BOOT/BOOTx64.efi
}
```

E no `config.plist`, deixe `Misc > Boot > ShowPicker = false` (segurar Esc mostra o menu do OpenCore). Depois rode `sudo update-grub`.

## Desenvolvimento iOS

- **Xcode 26.5 (Universal)** funciona. **Xcode 27+ é só para Apple Silicon** e não abre em Hackintosh ("bad CPU type").
- Simulador iOS 26.5 funciona, mas fica lento em HD mecânico. Use SSD.
- Homebrew não instala em hardware Intel/AMD no Tahoe. Instale Node, gh, Ruby etc. manualmente em `~/.local`.

## Limitações

- Sem virtualização dentro do macOS em AMD: nada de Docker, nem do emulador Android.
- AirDrop, Handoff e Continuidade limitados com o Wi-Fi Intel.
- Bluetooth Intel ainda não testado (IntelBluetoothFirmware + BlueToolFixup).
- Mapa USB ainda não feito.

## Créditos

[Acidanthera](https://github.com/acidanthera), [AMD-OSX](https://github.com/AMD-OSX), [ChefKiss (NootRX)](https://github.com/ChefKissInc/NootRX), [OpenIntelWireless](https://github.com/OpenIntelWireless), [Mieze (RealtekRTL8111)](https://github.com/Mieze/RTL8111_driver_for_OS_X) e o [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/).

---

## 🇺🇸 English summary

Working OpenCore 1.0.8 EFI for **macOS Tahoe 26.7.2** on Ryzen 5 5500 + A520M + RX 6600 + Intel AX210. Key points: NootRX for the RX 6600 (native driver panics), `DisableIoMapper = true` for itlwm on Tahoe, AMD Vanilla patches set for 6 cores, SMBIOS MacPro7,1 (**generate your own serials, they are blank here**). Xcode 26.5 works; Xcode 27+ is Apple Silicon only. If this helped you, leave a ⭐!

Use at your own risk.
