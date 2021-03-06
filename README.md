# 840g1-opencore-config
OpenCore configuration for my Elitebook.

# OpenCore version tested with: 0.5.6
## macOS version tested with: 10.15.3
Please do not use this with any newer or older version as some options may no longer exist or be interpreted differently, or new options may be needed in order to successfully boot.

# Specs
- System: HP Elitebook 840 G1 (board id 198F)
- CPU: i5 4300U
- GPU: Intel HD Graphics 4400
- RAM: 8 GB mixed 1600 MHz DDR3
- SSD: 256 GB Samsung 830
- Wifi: Broadcom BCM43224
- Ethernet: Intel I218LM
- Audio: IDT 92HD91BXX
- Card reader: Realtek RTS5227

# BIOS Settings (version 1.48)
Settings not specified are left on their default values or are unimportant.

- [ ] = unchecked
- [x] = checked

- Boot Options:
    - [ ] Fast Boot
    - [ ] All boot options except USB boot and Customized boot
    - [ ] Secure Boot
    - Boot mode: `UEFI Native (Without CSM)`
- Device Configurations
    - [ ] USB Legacy Support
    - [x] USB 3.0 (XHCI)
    - Video memory size: `64 MB`
    - [ ] Wake on USB
    - [x] VT-x
    - [ ] VT-d -> Could be left on, since the DisableIOMapper quirk is enabled in OpenCore.
    - Deep sleep: `Off`
- Built-In Device Options
    - [ ] Wireless Button State
    - [ ] LAN/WLAN switching
    - Wake on LAN: `Disable` -> System will instantly wake from sleep with this enabled.
- Port Options
    - [ ] Flash Media Reader -> Doesn't work anyway, see current issues below.

# OpenCore config noteworthy things

- USB on this machine is _weird_. Use the ports on the **left side of the machine** for installation, as these are directly routed to the Intel controller
- SIP is completely enabled
- `ig-platform-id` is set to `0x0A260006` and the device-id is spoofed to `0x0412` for the HD 4400 to work like HD 4600.
- `vault.plist` is *dis*abled for now.
- The system is set to play the startup beep/bong on its internal speaker. Drop the Resources folder from [OcBinaryData](https://github.com/acidanthera/OcBinaryData) in your EFI/OC folder. Strip out the excess files for languages you don't use for faster bootup times.
    - Lies! It doesn't currently do this. Keeping it here for future reference though.

## ScanPolicy enabled flags
ScanPolicy is set to a value of `2162947` or binary `001000010000000100000011`. This means the following flags are set:

- `OC_SCAN_FILE_SYSTEM_LOCK` (bit 0)
- `OC_SCAN_DEVICE_LOCK` (bit 1)
- `OC_SCAN_ALLOW_FS_APFS` (bit 8)
- `OC_SCAN_ALLOW_DEVICE_SATA` (bit 16)
- `OC_SCAN_ALLOW_DEVICE_USB` (bit 21)

Please refer to the OpenCore manual for the meaning of these flags. In short, these flags allow scanning of APFS partitions (for macOS) on SATA and USB drives only. The USB part should be temporary.

# ACPI edits
See the ACPI subfolder.

- `EHCx_OFF`: Turns off the EHC1 and EHC2 controllers, instead putting all devices on the USB3 controller
- `HPET`: Fixes some conflicting IRQs. Works together with a couple ACPI patches defined in config.plist
- `PLUG`: Sets `plugin-type` to 1, needed for AGPM and proper CPU power management.
- `PNLF`: Backlight patches, copied from WhateverGreen
- `UIAC`: SSDT defining which ports to keep enabled, for USB Inject All
    - *NOTE*: This does not include Bluetooth since my network card doesn't have bluetooth.

# Device Properties
- `PciRoot(0x0)/Pci(0x1b,0x0)`: IDT 92HD91BXX
    - `layout-id`: `84`, used to tell AppleALC which layout ID to pick to get the onboard sound working.
- `PciRoot(0x0)/Pci(0x2,0x0)`: Intel HD Graphics 4400
    - `ig-platform-id`: `0x0A260006` (reversed)
- `PciRoot(0x0)/Pci(0x1c,0x3)/Pci(0x0,0x0)`: Broadcom BCM43224
    - `brcmfx-country`: `NL` to make optimal use out of the Broadcom card and to reduce possible interference with surrounding devices.

# Drivers
Not included in this repo, I use the following drivers:

- [ApfsDriverLoader](https://github.com/acidanthera/AppleSupportPkg)
- [AudioDxe](https://github.com/acidanthera/AppleSupportPkg) TODO: Implement chime support
- FwRuntimeServices (in the OpenCore package)
- [HFSPlus](https://github.com/acidanthera/OcBinaryData/blob/master/Drivers/HfsPlus.efi)

# Kexts
Not included in this repo, I use the following kexts:

- [AirportBrcmFixup](https://github.com/acidanthera/AppleALC): Used to get the BCM43224 card working
- [AppleALC](https://github.com/acidanthera/AppleALC): Used to get the IDT onboard audio working
- [IntelMausi](https://github.com/acidanthera/IntelMausi): Driver for the built-in Ethernet controller
- [Lilu](https://github.com/acidanthera/Lilu): Requirement for most other kexts in this list
- [RTCMemoryFixup](https://github.com/acidanthera/RTCMemoryFixup): Fixup kext for the real time clock
- [USB Inject All](https://bitbucket.org/RehabMan/os-x-usb-inject-all/src/master/): Used for injecting only USB ports which are in use, to keep the system inside the 15 port limit
- [VirtualSMC](https://github.com/acidanthera/VirtualSMC): SMC emulator
- [VoodooPS2Controller](https://github.com/RehabMan/OS-X-Voodoo-PS2-Controller): Driver for the trackpad and keyboard
- [WhateverGreen](https://github.com/acidanthera/WhateverGreen): Required for proper functioning of both the integrated graphics card in headless/connectorless mode, and the RX580

# Current issues

## No battery status
There's some complicated EC/battery logic which has to be patched to get this working. See also [the laptop guide](https://fewtarius.gitbook.io/laptopguide/battery-power-management/locate-that-missing-battery).
NOTE: Loading the SMCBatteryManager kext does not cause a kernel panic, but also spams the kernel log with ACPI errors and does not give battery status as it is.

## "The CMOS checksum is invalid." error after reboot/boot
I've tried patching this in various areas to no avail. No idea what is causing this. Without RTCMemoryFixup macOS would reset the clock back to an earlier date. The OS thinks it's 1980, the BIOS says 2000. So it's possible the RTC as a whole doesn't play nice with macOS. Needs investigation nonetheless, this error is annoying.

## SD card reader doesn't work
It's a PCIe based card reader (RTS5227). Unlike USB card readers, this will not work out of the box.
Some drivers exist, the most recent one I could find being [Sinetek-rtsx](https://github.com/syscl/Sinetek-rtsx). 
However, that project seems dead and I have not (yet) tested this driver.

## Acidanthera VoodooPS2 doesn't play nice with the touchpad
The cursor is very fast, it almost feels like a touchscreen. No clicking support on any of the buttons. No idea what is causing this, however RehabMan's VoodooPS2Controller works well, so use that instead.

## Trackpoint doesn't work
I don't care, it's better this way. Probably requires an ACPI patch to get working.

## Wireless button is always orange
Minor "issue", not sure how I could fix this. Wifi works fine so I'm probably not going to bother.

## NVRAM is buggy
Understatement of the season. My machine had an old NVRAM setup from Clover. Clearing it via OpenCore, macOS, nor CleanNVRAM.efi did anything. Anything you specify in your OpenCore config.plist _will not_ override the stored variables.

What worked in the end was booting a Linux distro and manually deleting the NVRAM variables in `/sys/firmware/efi/efivars`. However, due to systemd magic, every non-standard variable, including Apple's, is created as an _immutable_ file. This however can be easily undone with `chattr -i [file]` after which you can delete them like normal. **BE SUPER CAREFUL AS THIS MAY BRICK YOUR MACHINE IF USED IMPROPERLY. I AM NOT KIDDING. THIS IS THE REASON IT IS MADE LIKE THIS IN THE FIRST PLACE.**