# Getting Started With OpenCore
A brief guide to using the OpenCore bootloader for hackintosh.

**This guide may not always be able to keep up with every change to OpenCore, (currently OpenCore is in active development, and therefore a moving target) please keep that in mind when compiling the latest version of OpenCore. To be safe, use release versions of OpenCore rather than the latest commits. ** This guide is intended to complement the excellent opencore "configuration.pdf" rather than be used instead of it. If you did not already do so, please read it now. https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/Configuration.pdf

# What is OpenCore?

OpenCore is an alternative bootloader to CloverEFI or Chameleon. It is not only for Hackintosh and can be used on real macs for purposes that require an emulated EFI. It also aims to have the ability to boot Windows and Linux. It has a clean codebase and aims to stay closer to how a real mac bootloader functions. While already functioning well, OpenCore should be considered in Alpha stage at this time and should be used by experienced hackintosh users and developers or users who are happy to recover a system which fails to boot, or becomes broken in some way.

# Current issues with OpenCore

* Z97 based systems require pure UEFI mode for booting (also known as Windows 8/10 mode).
* ~~Currently minimal support for emulated NVRAM (sorry z390 users, EmuVariableRuntimeDxe may work but won't save on reboot).~~ Support for nvram.plist has been added to OpenCore's most recent commit
* ~~FakeSMC sensors can't be injected, alternative is [HWSensors3](https://github.com/warexify/HWSensors3) or [VirtualSMC sensors](https://github.com/acidanthera/VirtualSMC).~~
* VoodooPS2Controller needs to be injected first, Keyboard second and Mouse/Trackpad third.
* NVMe issues if setup as a SATA device in BIOS.
* Sometimes can't access other partitions on the drive, solution is to "bless" the drive with Startup Disk

# Setting up OpenCore

Requirements:

* [OpenCorePkg](https://github.com/acidanthera/OpenCorePkg/releases) (Advanced users can build the latest from source code, less adavnced users should stick to the builds on the release page).
* [AppleSupportPkg](https://github.com/acidanthera/AppleSupportPkg/releases)
* [AptioFixPkg](https://github.com/acidanthera/AptioFixPkg/releases)
* [mountEFI](https://github.com/corpnewt/MountEFI) or some form of EFI mounting. Clover Configurator works just as well
* Xcode (or other plist editor) to edit .plist files.
* USB formatted as MacOS Journaled with GUID partition map. This is to test opencore without overwriting your working clover.
* Knowledge of how a hackintosh works and what files yours requires.
* A working Hackintosh to test on.

# Creating the USB

Creating the USB is simple. All you need to do is format it as MacOS Journaled with GUID partition map. There is no real size requirement for the USB as OpenCore's entire EFI is less than 5MB.

![Formatting the USB](https://i.imgur.com/9HNB1Jj.png)

Next we'll want to mount the EFI partition on the USB with either mountEFI or Clover Configurator.

![mountEFI](https://i.imgur.com/E6VM2Jc.png)

You'll notice that once we open the EFI partition, it's empty. This is where the fun begins.

![Empty EFI partition](https://i.imgur.com/EDeZB3u.png)

# Base folder structure

To setup OpenCore’s folder structure, you’ll want to grab those files from OpenCorePkg and construct your EFI to look like the one below:

![base EFI folder](https://i.imgur.com/lQ9u54w.png)

Now you can place your necessary .efi drivers from AppleSupportPkg and AptioFixPkg into the *drivers* folder and kexts/ACPI into their respective folders. Please note that UEFI drivers are not supported with OpenCore!

Here's what mine looks like (ignore my odd choice of kexts):

![Populated EFI folder](https://i.imgur.com/TLdovCj.png)

# Setting up your config.plist

Keep in mind with config.plist in OpenCore, they are different from Clover’s config.plist, they cannot be mixed and matched. It is not recomended to duplicate every patch and option from your clover config. 

First let’s duplicate the `sample.plist`, rename the duplicate to `config.plist` and open in Xcode.

![Base Config.plist](https://i.imgur.com/MklVb2Z.png)

At this point you've noticed there are a number of sections:

* ACPI: This is for loading, blocking and patching the ACPI.
* DeviceProperties: This is where you'd set PCI device patches like the Intel Framebuffer patch.
* Kernel: Where we tell OpenCore what kexts to load, what order to load and which to block.
* Misc: Settings for OpenCore's boot loader itself.
* NVRAM: This is where we set NVRAM properties like boot flags and SIP.
* Platforminfo: This is where we setup your SMBIOS.
* UEFI: UEFI drivers and related options. 

We can delete *#WARNING -1* and  *#WARNING -2* just to clean it up a bit.

# ACPI

**Add:** Here you add your SSDTs or custom DSDT. (SSDT-EC.aml for example)

**Block**: Certain systems benefit from dropping some acpi tables, most modern desktops however require nothing in this section.

**Patch**: In opencore we should be keeping ACPI patches to a minimum as they are often harmful and unecessary. If your system absolutely needs something, you should add it in this section.

**Quirk**: Certain ACPI fixes. Avoid unless needed.

* FadtEnableReset: NO (Enable reboot and shutdown on legacy hardware, not recommended unless needed)
* ~~IgnoreForWindows: NO (Disable ACPI modifications when booting Windows, only for those who made broken ACPI tables)~~ Removed from OpenCore
* NormalizeHeaders: Cleanup ACPI header fields, irrelevant in 10.14
* RebaseRegions: Attempt to heuristically relocate ACPI memory regions
* ResetHwSig: Needed for hardware that fail fail to maintain hardware signature across the reboots and cause issues with
waking from hibernation
* ResetLogoStatus: Workaround for systems running BGRT tables

![ACPI](https://i.imgur.com/sjlX3aT.png)

&#x200B;

# DeviceProperties

**Add**: Sets device properties from a map.

`PciRoot(0x0)/Pci(0x2,0x0)` -> `AAPL,ig-platform-id`

* Applies Framebuffer patch, insert required value from Framebuffer guide [here](https://www.insanelymac.com/forum/topic/334899-intel-framebuffer-patching-using-whatevergreen/?tab=comments#comment-2626271). Don't forget to add Stolemen and patch-enable.

`PciRoot(0x0)/Pci(0x1b,0x0)` -> `Layout-id`

* Applies AppleALC audio injection, insert required value from AppleALC documentation [here](https://github.com/acidanthera/AppleALC/wiki/Supported-codecs).

**Block**: Removes device properties from map (can delete, irrelevant for most users).

![DeviceProperties](https://i.imgur.com/8gujqhJ.png)

# Kernel

**Add**: Here's where you specify which kexts to load, order matters here so make sure Lilu.kext is always first! Other higher priority kexts come after Lilu such as, VirtualSMC, AppleALC, WhateverGreen, etc.

**Emulate**: Needed for spoofing unsupported CPUs like Pentiums and Celerons

* CpuidMask: When set to Zero, original CPU bit will be used
* CpuidData: The value for the CPU spoofing, don't forget to swap hex

**Block**: Blocks kexts from loading. Sometimes needed for disabling Apple's trackpad driver for some laptops.

**Patch**: Patches kexts (this is where you would add USB port limit patches and AMD CPU patches).

**Quirks**:

* AppleCpuPmCfgLock: Only needed when CFG-Lock can't be disabled in BIOS
* AppleXcpmCfgLock: Only needed when CFG-Lock can't be disabled in BIOS
* AppleXcpmExtraMsrs: Disables multiple MSR access needed for unsupported CPUs
* CustomSMBIOSGuid: Performs GUID patching for UpdateSMBIOSMode Custom mode. Usually relevant for Dell laptops
* DisbaleIOMapper: Needed to get around VT-D if unable to disable in BIOS, can interfere with Firmware
* ExternalDiskIcons: External Icons Patch, for when internal drives are treated as external drives
* LapicKernelPanic: Disables kernel panic on AP core lapic interrupt
* PanicNoKextDump: Allows for reading kernel panics logs when kernel panics occurs
* ThirdPartyTrim: Enables TRIM, not needed for AHCI or NVMe SSDs. It is better to enable third party trim via terminal command.
* XhciPortLimit: This is actually the 15 port limit patch, use only while you create a [USB map](https://usb-map.gitbook.io/project/) when possible. Its use is NOT recomended long term.

![Kernel](https://i.imgur.com/DcafUhE.png)

# Misc

**Boot**: Settings for boot screen (leave as-is unless you know what you're doing).
* Timeout: This sets how long OpenCore will wait until it automatically boots from the default selection
* ShowPicker: 
* UsePicker: Uses OpenCore's default GUI, set to NO if you wish to use a different GUI

**Debug**: Debug has special use cases, leave as-is unless you know what you're doing.
* DisableWatchDog: (May need to be set for yes if macOS is stalling on something while logging is enabled.

**Security**: Security is pretty self-explanatory.

* RequireSignature: We won't be dealing vault.plist so we can ignore
* RequireVault: We won't be dealing vault.plist so we can ignore as well
* ScanPolicy: Allows customization of disk and file system types which are scanned (and shown) by opencore at boot time.

**Tools** Used for running OC debugging tools like clearing NVRAM, we'll be ignoring this

![Misc](https://i.imgur.com/6NPXq0A.png)

# NVRAM

**Add**: 7C436110-AB2A-4BBB-A880-FE41995C9F82 (System Integrity Protection bitmask)

* boot-args: -v dart=0 debug=0x100 keepsyms=1 , etc (Boot flags)
* csr-active-config: <00000000> (Settings for SIP, recommeded to manully change this within Recovery partition with csrutil. 
   * `00000000` - SIP completely enabled
   * `30000000` - Allow unsigned kexts and writing to protected fs locations
   * `E7030000` - SIP completely disabled
* nvda_drv:  <> (For enabling Nvidia WebDrivers, set to 31 if running a [Maxwell or Pascal GPU](https://github.com/khronokernel/Catalina-GPU-Buyers-Guide/blob/master/README.md#Unsupported-nVidia-GPUs). This is the same as setting nvda_drv=1 but instead we translate it from [text to hex](https://www.browserling.com/tools/hex-to-text))
* prev-lang:kbd: <> (Needed for non-latin keyboards)

**Block**: Forcibly rewrites NVRAM variables, not needed for us as `sudo nvram` is prefered but useful for those edge cases

**LegacyEnable** Allows for NVRAM to be stored on nvram.plist 

**LegacySchema** Used for assigning nvram variable

![NVRAM](https://i.imgur.com/MPFj3TS.png)

# Platforminfo

**Automatic**: YES (Generates PlatformInfo based on Generic section instead of DataHub, NVRAM, and SMBIOS sections)

**Generic**:

* SpoofVendor: YES
* SystemUUID: Can be generated with MacSerial or use pervious from Clover's config.plist.
* MLB: Can be generated with MacSerial or use pervious from Clover's config.plist.
* ROM: <> (6 character MAC address, can be entirely random)
* SystemProductName: Can be generated with MacSerial or use pervious from Clover's config.plist.
* SystemSerialNumber: Can be generated with MacSerial or use pervious from Clover's config.plist.
`ROM must either be Apple ROM (dumped from a real Mac), or your NIC MAC address, or any random MAC address (could be just 6 random bytes)` - Vit9696

**DataHub**

**PlatformNVRAM**

**SMBIOS**

**UpdateDataHub**: YES (Update Data Hub fields)

**UpdateNVRAM**: YES (Update NVRAM fields)

**UpdateSMBIOS**: YES (Update SMBIOS fields)

**UpdateSMBIOSMode**: Create (Replace the tables with newly allocated EfiReservedMemoryType)

![PlatformInfo](https://i.imgur.com/dIKAlhj.png)

# UEFI

**ConnectDrivers**: YES (Forces .efi drivers, change to NO for faster boot times but cerain file system drivers may not load)

**Drivers**: Add your .efi drivers here.

**Protocols**:

* AppleBootPolicy: (Ensures APFS compatibility on VMs or legacy Macs)
* ConsoleControl: (Replaces Console Control protocol with a builtin version, needed for when firmware doesn’t support text output mode)
* DataHub: (Reinstalls Data Hub)
* DeviceProperties: (Ensures full compatibility on VMs or legacy Macs)

**Quirks**:

* ExitBootServicesDelay: 0 (Switch to 5 if running ASUS Z87-Pro with FileVault2)
* IgnoreInvalidFlexRatio: (Fix for when MSR_FLEX_RATIO (0x194) can't be disabled in the BIOS, required for all pre-skylake based systems)
* IgnoreTextInGraphics: (Fix for UI corruption when both text and graphics outputs happen)
* ProvideConsoleGop: (Enables GOP, AptioMemoryFix currently offers this but will soon be removed)
* ReleaseUsbOwnership: (Releases USB controller from firmware driver)
* RequestBootVarRouting: (Redirects AptioMemeoryFix from EFI_GLOBAL_VARIABLE_G to OC_VENDOR_VARIABLE_GUID. Needed for when firmware tries to delete boot entries)
* SanitiseClearScreen: (Fixes High resolutions displays that display OpenCore in 1024x768)

![UEFI](https://i.imgur.com/acZ1PUA.png)

# What your EFI should now look like:

![Finished EFI](https://i.imgur.com/TLdovCj.png)

# And now you're ready to boot!

![AboutThisMac](https://i.imgur.com/MFq1qGr.png)

# Making Opencore your Main Boot-Loader

So now you're ready to completely switch What you'll want to do is completely scrub your system of Clover. The main things to keep in mind is:
* Clover is on your Boot Drive (duh)
* Clover may be hiding in other spots (Clover Preference Pane and other tools that rely on Clover)

Cleaning up is actually quite simple, for your EFI you'll want to run [mountEFI](https://github.com/corpnewt/MountEFI), move Clover to somewhere safe(preferably a rescue USB) and copy OpenCore's EFI to the main drive's EFI partition. Certain system BIOS may require you to manually remove Clover as an EFI boot option (and extra special system might need a factory reset to permanently remove it)

Regarding apps that rely on Clover, you'll need to look through yourself but main culprit is Clover's Preference Pane which is used for updating Clover (I think you can see why that's an issue). You can find that at: `/Library/PreferencePanes/Clover.prefPane`.

# Credit
* [Apple](https://www.apple.com) for MacOS
* [vit9696](https://github.com/vit9696) for OpenCore and corrections for this guide
* [InsanelyMac's OpenCore forums](https://www.insanelymac.com/forum/topic/338516-opencore-discussion/) for finding issues with hardware and their work arounds
* [icedterminal](https://github.com/icedterminal) for heavy grammar correction and Clover Removal
