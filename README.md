# Asus Zenbook 13 (UX325UM - UM325U)
![asus-zenbook-14_um325ua-kg189w_06](https://github.com/anerik70/Asus-Zenbook-13-UM325U/assets/5209138/6566b8de-25c6-4ec8-b32f-beb3cf952c54)
![Ryzentosh UX325U - UM325U](https://github.com/anerik70/Asus-Zenbook-13-UM325U/assets/5209138/b950c14e-583e-4843-9225-a2bf4789815a)


# SYSTEM
- Processor:	AMD Ryzen 5 5500U (APU Mobile).
- Memory:	16GB.
- Graphics:	AMD Radeon RX Vega 10 (Renoir).
- Wireless Network:	Intel AX200.
- Strorage:	ASUS NVME (not works in mac) replace by KingSpec NVME from aliexpress https://a.aliexpress.com/_mNrDMoO
- Audio:	Realtek ALC294.
- Camera: HD camera with IR function compatible with Windows Hello.
- TouchPad: ELAN I2C
- Realtek Card Reader

# Installation Instructions

1. Setup bios: Disable fast boot, disable secure boot
2. Replace NVME.
3. Disable kext "NootedRed.kext". Enable again after installing MacOS

# Post Install Instructions

1. Go to System preferences/Displays and disble HDR (Hight Dinamic Range) to fix the flickering display
2. Enable "NootedRed.kext" in config.plist
3. Restart
4. Use Lunar.app for brightness control (setup softwe dimming)
![Captura de pantalla 2023-09-15 a la(s) 5 34 11 p  m](https://github.com/anerik70/Asus-Zenbook-13-UM325U/assets/5209138/37c5cb76-401c-4333-aa0d-553d263f8572)

# What's working:

- Wi-Fi/Bluetooth (Intel AX200)
- USB Ports, USB-C
- Keyboard
- Trackpad I2C
- Audio
- Internal Microphone: working with bad quality using AMDmicrophone.kext fork of https://github.com/qhuyduong/AMDMicrophone
- Graphics acceleration (Shaneee's patch)
- Control brightness, sound and touchpad via keyboard keys
- Battery Status
- Built-In Camera
- CPU PowerManagement
- Monitor CPU, GPU temperature
- HDMI
- Card Reader

# What's Not Working:
- Sleep
- HDR
- Native brightness

# Credits:
- Apple for macOS
- AMD-OSX Developers for kernel patches for AMD CPUs
- Acidanthera for OpenCore and most of used kexts
- Trulyspinach for Ryzen power management and monitoring kexts
- DhinakG for USBToolBox
- Dortania for OpenCore configuration guides
- AMD-OSX Community for support while making my Hackintosh
