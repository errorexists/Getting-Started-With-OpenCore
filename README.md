# Getting Started With OpenCore
A brief guide to using the OpenCore bootloader for hackintosh.

**This guide may not always be able to keep up with every change to OpenCore, (currently OpenCore is in active development, and therefore a moving target) please keep that in mind when compiling the latest version of OpenCore. To be safe, use release versions of OpenCore rather than the latest commits. ** This guide is intended to complement the excellent opencore "configuration.pdf" rather than be used instead of it. If you did not already do so, please read it now: https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/Configuration.pdf

# What is OpenCore?

OpenCore is an alternative bootloader to CloverEFI or Chameleon. It is not only for Hackintosh and can be used on real macs for purposes that require an emulated EFI. It also aims to have the ability to boot Windows and Linux. It has a clean codebase and aims to stay closer to how a real mac bootloader functions. Kext injection is greatly improved. While already functioning well, OpenCore should be considered in Alpha stage at this time and should be used by experienced hackintosh users and developers or users who are happy to recover a system which fails to boot, or becomes broken in some way.

# Current issues with OpenCore

* Z97 based systems require pure UEFI mode for booting (also known as Windows 8/10 mode).
* Z390 based systems require workarounds to non working NVRAM.
* VoodooPS2Controller needs to be injected first, Keyboard second and Mouse/Trackpad third.
* NVMe issues if setup as a SATA device in BIOS.
* Sometimes can't access other partitions on the drive, solution is to "bless" the drive with Startup Disk.

# Setting up OpenCore

Requirements:

* [OpenCorePkg](https://github.com/acidanthera/OpenCorePkg/releases) (Advanced users can build the latest from source code, less advanced users should stick to the builds on the release page).
* [AppleSupportPkg](https://github.com/acidanthera/AppleSupportPkg/releases)
* [AptioFixPkg](https://github.com/acidanthera/AptioFixPkg/releases)
* [mountEFI](https://github.com/corpnewt/MountEFI) or some form of EFI mounting. Clover Configurator works just as well
* Xcode (or other plist editor) to edit .plist files.
* USB formatted as MacOS Journaled with GUID partition map. This is to test opencore without overwriting your working Clover.
* Knowledge of how a hackintosh works and what files yours requires.
* A working Hackintosh to test on.
* Time and patience. Without these, you are wasting your effort. 

# Creating the USB

Creating the USB is simple, format a stick as MacOS Journaled with GUID partition map. There is no real size requirement for the USB as OpenCore's entire EFI is less than 5MB.

![Formatting the USB](https://i.imgur.com/9HNB1Jj.png)

Next we'll want to mount the EFI partition on the USB with either mountEFI or Clover Configurator.

![mountEFI](https://i.imgur.com/E6VM2Jc.png)

You'll notice that once we open the EFI partition, it's empty.

![Empty EFI partition](https://i.imgur.com/EDeZB3u.png)

# Base folder structure

To setup OpenCore’s folder structure, you’ll want to grab those files from OpenCorePkg and construct your EFI to look like the one below:

![base EFI folder](https://i.imgur.com/lQ9u54w.png)

Now you can place your necessary .efi drivers from AppleSupportPkg and AptioFixPkg into the *drivers* folder and kexts/ACPI into their respective folders. Please note that UEFI drivers are not supported with OpenCore!

Here's what mine looks like:

![Populated EFI folder](https://i.imgur.com/TLdovCj.png)

# Setting up your config.plist

Keep in mind with config.plist in OpenCore, they are different from Clover’s config.plist, they cannot be mixed and matched. It is not recommended to duplicate every patch and option from your clover config. 

First let’s duplicate the `sample.plist`, rename the duplicate to `config.plist` and open in Xcode.

![Base Config.plist](https://i.imgur.com/MklVb2Z.png)

The config contains a number of sections:

* ACPI: This is for loading, blocking and patching the ACPI.
* DeviceProperties: This is where you'd set PCI device patches like the Intel Framebuffer patch.
* Kernel: Where we tell OpenCore what kexts to load, what order to load and which to block.
* Misc: Settings for OpenCore's boot loader itself.
* NVRAM: This is where we set NVRAM properties like boot flags and SIP.
* Platforminfo: This is where we setup your SMBIOS.
* UEFI: UEFI drivers and related options. 

We can delete *#WARNING -1* and  *#WARNING -2* You did heed the warning didn't you?

# ACPI

**Add:** Here you add your SSDTs or custom DSDT. (SSDT-EC.aml for example)

**Block**: Certain systems benefit from dropping some acpi tables, most modern desktops however require nothing in this section.

**Patch**: In opencore we should be keeping ACPI patches to a minimum as they are often harmful and unnecessary. If your system absolutely needs something, you should add it in this section.

**Quirk**: Certain ACPI fixes. Avoid unless necessary.

* FadtEnableReset: NO (Enable reboot and shutdown on legacy hardware, not recommended unless needed)
* NormalizeHeaders: Cleanup ACPI header fields, irrelevant in 10.14
* RebaseRegions: Attempt to heuristically relocate ACPI memory regions
* ResetHwSig: Needed for hardware that fail to maintain hardware signature across the reboots and cause issues with
waking from hibernation
* ResetLogoStatus: Workaround for systems running BGRT tables

![ACPI](https://imgur.com/WaGa6PT.png)

&#x200B;

# DeviceProperties

**Add**: Sets device properties from a map.

`PciRoot(0x0)/Pci(0x2,0x0)` -> `AAPL,ig-platform-id`

* Applies Framebuffer patch, insert required value from Framebuffer guide [here](https://www.insanelymac.com/forum/topic/334899-intel-framebuffer-patching-using-whatevergreen/?tab=comments#comment-2626271). Don't forget to add Stolemem and patch-enable.

`PciRoot(0x0)/Pci(0x1b,0x0)` -> `Layout-id`

* Applies AppleALC audio injection, insert required value from AppleALC documentation [here](https://github.com/acidanthera/AppleALC/wiki/Supported-codecs).

**Block**: Removes device properties from map (can delete, irrelevant for most users).

![DeviceProperties](https://i.imgur.com/8gujqhJ.png)

# Kernel

**Add**: Here's where you specify which kexts to load, and the order in which they are loaded, Lilu.kext should be first!
Plugins for other kexts should always come after the main kext. Lilu plugins- after Lilu, VirtualSMC plugins- after VirtualSMC etc.

**Emulate**: Needed for spoofing unsupported CPUs like Pentiums and Celerons

* CpuidMask: When set to Zero, original CPU bit will be used
* CpuidData: The value for the CPU spoofing, don't forget to swap hex

**Block**: Blocks kexts from loading. Sometimes needed for disabling Apple's trackpad driver for some laptops.

**Patch**: Patches kexts (this is where you would add USB port limit patches and AMD CPU patches).

**Quirks**:

* AppleCpuPmCfgLock: Only needed when CFG-Lock can't be disabled in BIOS. Avoid unless necessary.
* AppleXcpmCfgLock: Only needed when CFG-Lock can't be disabled in BIOS. Avoid unless necessary.
* AppleXcpmExtraMsrs: Disables multiple MSR access needed for unsupported CPUs.
* CustomSMBIOSGuid: Performs GUID patching for UpdateSMBIOSMode Custom mode. Usually relevant for Dell laptops.
* DisbaleIOMapper: Preferred to dropping DMAR in ACPI section or disabling VT-D in bios.
* ExternalDiskIcons: External Icons Patch, for when internal drives are treated as external drives
* LapicKernelPanic: Disables kernel panic on AP core lapic interrupt. Often needed on HP laptops.
* PanicNoKextDump: Allows for reading kernel panics logs when kernel panics occurs.
* ThirdPartyTrim: Enables TRIM, not needed for AHCI or NVMe SSDs. It is better to enable third party trim via terminal command trimforce.
* XhciPortLimit: This is actually the 15 port limit patch, use only while you create a [USB map](https://usb-map.gitbook.io/project/) when possible. Its use is NOT recomended long term.

![Kernel](https://i.imgur.com/DcafUhE.png)

# Misc

**Boot**: Settings for boot screen.
* Timeout: This sets how long OpenCore will wait until it automatically boots from the default selection
* ShowPicker: If you need to see the picker screen, you better choose YES.
* UsePicker: Uses OpenCore's default GUI, set to NO if you wish to use a different GUI.
* Target: Setting for logging type (by default logging output is hidden).

**Debug**: Debug has special use cases, leave as-is unless you know what you're doing.
* DisableWatchDog: (May need to be set to yes if macOS is stalling while logging to file is enabled).

**Security**:

* RequireSignature: We won't be dealing vault.plist so we can ignore.
* RequireVault: We won't be dealing vault.plist so we can ignore as well.
* ScanPolicy: Allows customization of disk and file system types which are scanned (and shown) by opencore at boot time.

**Tools** Used for running OC debugging tools like clearing NVRAM, we'll be ignoring this.

![Misc](https://i.imgur.com/6NPXq0A.png)

# NVRAM

**Add**: 7C436110-AB2A-4BBB-A880-FE41995C9F82 (System Integrity Protection bitmask)

* boot-args: -v dart=0 debug=0x100 keepsyms=1 , etc (Boot flags)
* csr-active-config: <00000000> (Settings for SIP, recommeded to manully change this within Recovery partition with csrutil. 
   * `00000000` - SIP completely enabled
   * `30000000` - Allow unsigned kexts and writing to protected fs locations
   * `E7030000` - SIP completely disabled
* nvda_drv:  <> (For enabling Nvidia WebDrivers, set to 31 if running a [Maxwell or Pascal GPU](https://github.com/khronokernel/Catalina-GPU-Buyers-Guide/blob/master/README.md#Unsupported-nVidia-GPUs). This is the same as setting nvda_drv=1 but instead we translate it from [text to hex](https://www.browserling.com/tools/hex-to-text))
* prev-lang:kbd: <> (Needed for non-latin keyboards) If you find Russian, you didnt read the manual...

**Block**: Forcibly rewrites NVRAM variables, not needed for us as `sudo nvram` is prefered but useful for those edge cases.

**LegacyEnable** Allows for NVRAM to be stored on nvram.plist for systems without working NVRAM.

**LegacySchema** Used for assigning nvram variable

![NVRAM](https://i.imgur.com/MPFj3TS.png)

# Platforminfo

**Automatic**: YES (Generates PlatformInfo based on Generic section instead of DataHub, NVRAM, and SMBIOS sections).

**Generic**:

* SpoofVendor: YES (This prevents issues with having "Apple.inc" as manufacturer).
* SystemUUID: Can be generated with MacSerial or use previous from Clover's config.plist.
* MLB: Can be generated with MacSerial or use previous from Clover's config.plist.
* ROM: <> (6 character MAC address, can be entirely random but should be unique).
* SystemProductName: Can be generated with MacSerial or use previous from Clover's config.plist.
* SystemSerialNumber: Can be generated with MacSerial or use previous from Clover's config.plist.

**DataHub**

**PlatformNVRAM**

**SMBIOS**

**UpdateDataHub**: YES (Update Data Hub fields)

**UpdateNVRAM**: YES (Update NVRAM fields)

**UpdateSMBIOS**: YES (Update SMBIOS fields)

**UpdateSMBIOSMode**: Create (Replace the tables with newly allocated EfiReservedMemoryType)

![PlatformInfo](https://i.imgur.com/dIKAlhj.png)

# UEFI

**ConnectDrivers**: YES (Forces .efi drivers, change to NO for faster boot times but certain file system drivers may not load)

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
* RequestBootVarRouting: (Redirects AptioMemoryFix from EFI_GLOBAL_VARIABLE_G to OC_VENDOR_VARIABLE_GUID. Needed for when firmware tries to delete boot entries)
* SanitiseClearScreen: (Fixes High resolutions displays that display OpenCore in 1024x768)

![UEFI](https://i.imgur.com/acZ1PUA.png)

# What your EFI should now look like:

![Finished EFI](https://i.imgur.com/TLdovCj.png)

# And now you are ready to boot!

![AboutThisMac](https://i.imgur.com/MFq1qGr.png)

# Making Opencore your Main Bootloader

When you are happy opencore boots your system correctly, simply mount your Clover efi partition, (back it up somewhere safe) and overwrite it with your OpenCore one. Certain system BIOS may require you to manually remove Clover as an EFI boot option (and extra special system might need a factory reset to permanently remove it).

Remove Clover's Preference Pane (if installed) You can find that at: `/Library/PreferencePanes/Clover.prefPane`.

# Credit
* [Apple](https://www.apple.com) for MacOS
* [vit9696](https://github.com/vit9696) for OpenCore and corrections for this guide
* [InsanelyMac's OpenCore forums](https://www.insanelymac.com/forum/topic/338516-opencore-discussion/) for finding issues with hardware and their work arounds
* [icedterminal](https://github.com/icedterminal) for heavy grammar correction and Clover Removal
* [khronokernel](https://github.com/khronokernel) for the lovely email. :D
* [MacFriedIntel](https://github.com/MacFriedIntel) for not giving a shit :D
