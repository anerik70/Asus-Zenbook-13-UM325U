# ASUS ZenBook 13 OLED UM325U (Ryzen 5 5500U) — macOS Tahoe 26.7.3 Hackintosh

![asus-zenbook-14_um325ua-kg189w_06](https://github.com/anerik70/Asus-Zenbook-13-UM325U/assets/5209138/6566b8de-25c6-4ec8-b32f-beb3cf952c54)

A working Ryzen (Cezanne) laptop Hackintosh on **macOS Tahoe 26.7.3 (25G416)**, updated November 2026. The headline result: **native OLED brightness control works** — solved through the AMD DisplayPort AUX backlight interface with [AuxDim](https://github.com/nickk19/AuxDim) 0.3.0, no software dimming needed. Sleep remains the one unsolved front, and this page documents exactly where and why it fails, so nobody else has to re-derive it.

> This repo predates Tahoe (it started in 2023 on an older macOS). Everything below describes the **current** build. Hardware notes that changed since the first version of this README: the iGPU is a **Vega 7** (Cezanne, not Renoir/Vega 10), the audio codec is an **ALC295**, and the SSD is now a **WD_BLACK SN850X 1 TB**.

## Hardware

| Part | Detail |
|---|---|
| CPU | AMD Ryzen 5 5500U (Cezanne) |
| RAM | 16 GB LPDDR4X |
| GPU | AMD Radeon RX Vega 7 (0x164C), via NootedRed |
| Display | Samsung OLED **ATNA33XC11-0**, 13.3" |
| Wi-Fi / BT | Intel AX200 |
| Audio | Realtek ALC295 |
| Trackpad | ELAN1206, I2C |
| Keyboard | PS2 |
| Card reader | Realtek (PCIe, rtsx) |
| Storage | WD_BLACK SN850X 1 TB (stock ASUS NVMe replaced) |
| Camera | HD USB webcam (IR, Windows Hello) |

Multiboot on the same SSD: macOS + Windows + Kali Linux, each with its own EFI partition.

## Software stack

- **OpenCore 1.0.8**, SMBIOS `MacBookPro16,2`
- **Lilu 1.7.1**
- **NootedRed 0.9.0** (iGPU acceleration, Metal works)
- **AirportItlwm 2.4.1** (Wi-Fi; no OCLP root patches — they were reverted and are not used)
- **AppleALC 1.9.5 + SimpleHDA**, layout `alcid=44`
- **Custom USB map** built in Tahoe with corpnewt/USBMap (controllers XHC0 + XHC2; camera and Bluetooth ports marked internal, connector 255)
- **Intel Bluetooth**: IntelBluetoothFirmware + IntelBTPatcher 2.5.2 (lshbluesky fork) + BlueToolFixup 2.7.2
- **AuxDim 0.3.0** — OLED brightness through the AMD AUX interface (details below)
- **AMDMicrophone** for the internal mic (installed in /Library/Extensions)

Production boot-args:

```
-v amfi=0x80 -ibtcompatbeta debug=0x144 keepsyms=1 alcid=44 revpatch=sbvmm,cpuname itlwm_cc=CO -lilubetaall -auxdimamd
```

(The `-v debug=0x144 keepsyms=1` trio is debugging leftovers I keep on purpose; a clean setup can drop it.)

## ⭐ Native OLED brightness — SOLVED (AuxDim, AMD backend)

This was the longest-running problem on this machine. On Tahoe the classic PWM backlight path is dead: the slider exists, macOS sends the requests (PNLF/AppleBacklight all present), and the panel ignores them. HDR mode *does* change luminance but flickers (see Known issues), so for over a year the practical answer was software dimming with Lunar.

The fix came from [AuxDim](https://github.com/nickk19/AuxDim), a Lilu plugin that dims OLED panels over the DisplayPort AUX channel. What we found on this panel, and what may help other Cezanne/OLED owners:

- NootedRed's framebuffer **does** publish a native AUX interface on AMD (`native-aux=yes`) — first confirmed AMD data point for the project.
- The panel advertises **both** backlight interfaces, but ASUS firmware enables neither the Intel nits mode (DPCD `0x344 = 0x00`) nor the VESA mode (`0x720` bit 0 = 0).
- Live DPCD dumps taken in Linux (Kali, `/dev/drm_dp_aux0`) at ~20% and 100% brightness proved the panel is actually driven through **AMD's own nits interface**: enable `0x32F = 0x01`, level at `0x320` in **millinits** (80,000 → 400,000), readback echoed at `0x326`/`0x32A`, transition time at `0x324` (50 ms), control at `0x32E`. The Intel level register `0x354` tracks brightness too (80 → 400); the VESA block `0x700` never changes.

With those dumps, nickk19 built an AMD backend in three days. Since **AuxDim 0.3.0**:

- boot-arg **`-auxdimamd`** and the macOS slider (and Fn keys) drive the panel directly over AUX — real panel dimming, not software darkening.
- The lowest slider position turns the panel fully dark (default floor is 0 nits; `auxdimminnits=N` keeps a visible minimum if you prefer).
- Brightness is retained across display sleep/wake (verified: hundreds of AUX writes, zero failures).

Full story, probe output and dumps: [AuxDim issue #1](https://github.com/nickk19/AuxDim/issues/1) — this machine was the first AMD test case. AuxDim 0.4.0 (November 2026) adds macOS-style half-dimming right before display sleep; 0.3.0 is the version verified here.

## Bluetooth — working (with a specific recipe)

Intel Bluetooth on Tahoe took the longest detour on this build: with several firmware/patcher combinations the firmware uploaded to the AX200 but Apple's `IOBluetoothHostControllerUSBTransport` never registered, leaving a ghost controller (address NULL, firmware v0). What finally made it pair and stay paired:

1. Kext order at the **end** of Kernel → Add: **IntelBluetoothFirmware → BlueToolFixup → IntelBTPatcher** (BlueToolFixup 2.7.2, firmware/patcher 2.5.2 from the lshbluesky fork).
2. The two Bluetooth variables present in **NVRAM → Add**, zeroed (`bluetoothExternalDongleFailed`, `bluetoothInternalControllerInfo`).
3. A preferences cleanup: `sudo pkill bluetoothd`, then remove `com.apple.bluetooth*` from `/Library/Preferences`, then reboot.

Individually, none of the three was enough (the plist cleanup alone had been tried weeks earlier with no effect); the combination was. One more warning from this machine's history: a corrupted `/usr/sbin/bluetoothd` binary (APFS compressed-data hash mismatch, visible in the crash report's ktriage info) produces pairing crashes that look exactly like a kext problem. If pairing kills Bluetooth, read the `.ips` before changing kexts.

## Sleep — NOT working, and here is the whole map

Sleep is the one open front. Everything below was tested and measured on this machine:

- **By default the firmware exposes only S0 Modern Standby + Hibernate** (confirmed with `powercfg /a` in Windows: S3 "the system firmware does not support this standby state"). macOS sleeps via S3, so out of the box the machine only turns its screen off.
- **S3 can be enabled in the BIOS.** With S3 on, macOS enters *real* sleep (breathing LED) — and **reboots on wake, with no panic and no "Sleep Wake failure in EFI" report**. The DSDT shows why this is a firmware story: the S3 package exists but is hidden as `Name (XS3, …)` and is only published as `\_S3` when the BIOS option is on; `_WAK`/`_PTS` are complete. The resume dies inside the firmware's S3 path, before the kernel regains control. (Side effect: with S3 enabled in BIOS, Windows does not boot.)
- **Hibernation routes are dead ends here**: `hibernatemode 25` is silently ignored by the platform (image created, never used, no sleep entry in the log), and HibernationFixup 1.5.4 changes nothing — the sleep request dies before the kext can help. `darkwake=0` also makes no difference.
- **Deep Idle (S0ix) cannot work on AMD under macOS.** [S0ixStandby](https://github.com/labrouss/S0ixStandby) (SSDT-DEEPIDLE + kext) was installed and tested: this machine's PEP (`\_SB.PEP`, AMDI0005) is present with the right LPS0 _DSM surface (Microsoft mask 0x1F9), Deep Idle engaged, the LED breathed — and wake hung with the CPU halted, identically with `-s0ixnodsm` (no _DSM calls at all), which clears the kext/firmware exchange. The project's author confirmed the root cause: on the first Deep Idle, XNU halts the CPU through a path only Intel's XCPM handles, so on AMD it never comes back. His v0.5.0 ships an "AMD guard" based on this machine's report: [S0ixStandby issue #2](https://github.com/labrouss/S0ixStandby/issues/2).

Applied and kept (harmless, and required if a future route appears): `pmset` recipe — `hibernatemode 0 standby 0 autopoweroff 0 powernap 0 proximitywake 0 tcpkeepalive 0 womp 0 ttyskeepawake 0`.

**Practical upshot for other Ryzen laptop owners:** if your firmware hides S3 and only offers Modern Standby, expect exactly this wall on macOS. Shut down for real rest; display sleep is not sleep.

## What's working

- Graphics acceleration (Metal), HDMI
- **Native brightness** (AuxDim, AMD AUX interface) — slider + Fn keys
- Wi-Fi (AirportItlwm), Bluetooth (pairing included)
- Audio out (speakers), keyboard backlight keys, sound keys
- Trackpad I2C with gestures, PS2 keyboard
- USB ports (USB-A + both USB-C), custom map
- Built-in camera
- Battery status, CPU power management, temperature monitoring
- Card reader (microSD)

## What's not working / known issues

- **Sleep** (see the full map above)
- **HDR flicker**: with HDR enabled the panel flickers — a known NootedRed driver trait, not panel damage. Tracked upstream: [NootedRed #301](https://github.com/ChefKissInc/NootedRed/issues/301) and [#379](https://github.com/ChefKissInc/NootedRed/issues/379); my own earlier report is issue #2 in this repo. **Keep HDR off**; with AuxDim, SDR brightness control is complete anyway.
- **Internal microphone**: works via AMDMicrophone but with poor quality — a known Cezanne ceiling, not a configuration error.
- **macOS 27 will never run on this machine**: it is Apple-Silicon-only, with no x86 build at all. Tahoe 26 is the end of the line for x86 Hackintoshes; expect security updates for roughly three more years.

## BIOS setup

- Disable Fast Boot and Secure Boot
- Leave the S3 option **off** (default) unless you are experimenting — with it on, macOS reboots on wake and Windows won't boot

## Method note

Every EFI change on this machine was made one at a time, with a full verified backup of the EFI first, a bootable rescue USB at hand, and a fingerprint check of the target EFI partition before writing (this laptop's disk numbering changes between boots, and an external drive's ESP can look identical — ask me how I know). If you adapt anything from this page, keep the same discipline.

## Credits

- Apple for macOS
- AMD-OSX developers for the AMD kernel patches
- Acidanthera for OpenCore and most kexts used here
- [nickk19](https://github.com/nickk19) for AuxDim and for building the AMD backend from this machine's DPCD dumps
- [labrouss](https://github.com/labrouss) for S0ixStandby and for documenting the AMD Deep Idle verdict
- ChefKissInc for NootedRed
- Trulyspinach for Ryzen power management and monitoring
- DhinakG / corpnewt for USBToolBox / USBMap
- Dortania for the OpenCore guides
- The AMD-OSX community
