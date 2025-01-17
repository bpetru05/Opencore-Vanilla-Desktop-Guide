# Haswell

### Starting Point

You'll want to start with the sample.plist that OpenCorePkg provides you and rename it to config.plist. Next, open up your favourite XML editor like Xcode and we can get to work.

### ACPI

![ACPI](https://i.imgur.com/ByHZn1D.png)

The above ACPI patch is only an example, please read below for more info

**Add:**

This is where you'll add SSDT patches for your system, these are most useful for laptops and OEM desktops but also common for [USB maps](https://usb-map.gitbook.io/project/), [disabling unsupported GPUs](https://khronokernel-4.gitbook.io/disable-unsupported-gpus/) and such

**Block**

This drops certain ACPI tabes from loading, for us we can ignore this

**Patch**:

This section allows us to dynamically modify parts of the ACPI \(DSDT, SSDT, etc.\) via OpenCore. macOS usually does not care much about ACPI, so in the majority of the cases, you need to do nothing here. For those who need DSDT patches for things like EHCI controllers can utilize the [SSDT-EHCx\_ODD.dsl](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/AcpiSamples/SSDT-EHCx_OFF.dsl) or use similar Device Property patching like what's seen with Framebuffer patching.

And to grab the location of such devices can use [gfxutil](https://github.com/acidanthera/gfxutil/releases).

**Quirk**: Settings for ACPI.

* **FadtEnableReset**: NO
  * Enable reboot and shutdown on legacy hardware, not recommended unless needed
* **NormalizeHeaders**: NO
  * Cleanup ACPI header fields, only relevant for macOS High Sierra 10.13
* **RebaseRegions**: NO
  * Attempt to heuristically relocate ACPI memory regions, not needed unless custom DSDT is used.
* **ResetHwSig**: NO
  * Needed for hardware that fail to maintain hardware signature across the reboots and cause issues with waking from hibernation
* **ResetLogoStatus**: NO
  * Workaround for OEM Windows logo not drawing on systems with BGRT tables.

### Booter

![Booter](https://i.imgur.com/IRSYAtQ.png)

This section is dedicated to quirks relating to FwRuntimeServices.efi, the replacement for AptioMemoryFix.efi

**Quirks**:

* **AvoidRuntimeDefrag**: YES
  * Fixes UEFI runtime services like date, time, NVRAM, power control, etc
* **DisableVariableWrite**: NO
  * Needed for systems with non-functioning NVRAM like Z390 and such
* **DiscardHibernateMap**: NO
  * Reuse original hibernate memory map, only needed for certain legacy hardware 
* **EnableSafeModeSlide**: YES
  * Allows for slide values to be used in Safemode
* **EnableWriteUnprotector**: YES
  * Removes write protection from CR0 register during their execution
* **ForceExitBootServices**: NO
  * Ensures ExitBootServices calls succeeds even when MemoryMap has changed, don't use unless necessary\) 
* **ProtectCsmRegion**: NO
  * Needed for fixing artifacts and sleep-wake issues, AvoidRuntimeDefrag resolves this already so avoid this quirk unless necessary
* **ProvideCustomSlide**: YES
  * If there's a conflicting slide value, this option forces macOS to

    use a pseudo-random value. Needed for those receiving `Only N/256 slide values are usable!` debug message
* **SetupVirtualMap**: YES
  * Fixes SetVirtualAddresses calls to virtual addresses
* **ShrinkMemoryMap**: NO
  * Needed for systems with large memory maps that don't fit, don't use unless necessary

### DeviceProperties

![DeviceProperties](https://i.imgur.com/vgLIhOo.png)

**Add**: Sets device properties from a map.

This section is set up via Headkaze's [_Intel Framebuffer Patching Guide_](https://www.insanelymac.com/forum/topic/334899-intel-framebuffer-patching-using-whatevergreen/?tab=comments#comment-2626271) and applies only one actual property to begin, which is the _ig-platform-id_. The way we get the proper value for this is to look at the ig-platform-id we intend to use, then swap the pairs of hex bytes.

If we think of our ig-plat as `0xAABBCCDD`, our swapped version would look like `DDCCBBAA`

The two ig-platform-id's we use are as follows:

* `0x0D220003` - this is used when the iGPU is used to drive a display
  * `0300220D` when hex-swapped
* `0x04120004` - this is used when the iGPU is only used for compute tasks and doesn't drive a display
  * `04001204` when hex-swapped

I added another portion as well that shows a `device-id` fake in case you have an HD 4400 which is unsupported in macOS.

For this - we follow a similar procedure as our above ig-platform-id hex swapping - but this time, we only work with the first two pairs of hex bytes. If we think of our device id as `0xAABB0000`, our swapped version would look like `0xBBAA0000`. We don't do anything with the last 2 pairs of hex bytes.

The device-id fake is set up like so:

* `0x04120000` - this is the device id for HD 4600 which does have support in macOS
  * `12040000` when hex swapped

`PciRoot(0x0)/Pci(0x1b,0x0)` -&gt; `Layout-id`

* Applies AppleALC audio injection, you'll need to do your own research on which codec your motherboard has and match it with AppleALC's layout. [AppleALC Supported Codecs](https://github.com/acidanthera/AppleALC/wiki/Supported-codecs).

Keep in mind that some motherboards have different device locations, you can find yours by either examining the device tree in IOReg or using [gfxutil](https://github.com/acidanthera/gfxutil/releases)

Layout=5 would be interpreted as `05000000`

**Block**: Removes device properties from map, for us we can ignore this

### Kernel

![Kernel](https://i.imgur.com/2DkdqCv.png)

**Add**: Here's where you specify which kexts to load, order matters here so make sure Lilu.kext is always first! Other higher priority kexts come after Lilu such as VirtualSMC, AppleALC, WhateverGreen, etc.

**Emulate**: Needed for spoofing unsupported CPUs like Pentiums and Celerons

* CpuidMask: When set to Zero, original CPU bit will be used

  `<Clover_FCPUID_Extended_to_4_bytes_Swapped_Bytes> | 00 00 00 00 | 00 00 00 00 | 00 00 00 00`

* CpuidData: The value for the CPU spoofing

  `FF FF FF FF | 00 00 00 00 | 00 00 00 00 | 00 00 00 00`

**Block**: Blocks kexts from loading. Sometimes needed for disabling Apple's trackpad driver for some laptops.

**Patch**: Patches kexts \(this is where you would add newer USB port limit patches and AMD CPU patches. Do note that the XhciPortLimit quirk is preferred for USB port limit patches\).

**Quirks**:

* **AppleCpuPmCfgLock**: NO 
  * Only needed when CFG-Lock can't be disabled in BIOS, Clover counterpart would be AppleICPUPM
* **AppleXcpmCfgLock**: NO 
  * Only needed when CFG-Lock can't be disabled in BIOS, Clover counterpart would be KernelPM
* **AppleXcpmExtraMsrs**: NO 
  * Disables multiple MSR access needed for unsupported CPUs like Pentiums and certain Xeons
* **CustomSMBIOSGuid**: NO 
  * Performs GUID patching for UpdateSMBIOSMode Custom mode. Usually relevant for Dell laptops
* **DisableIOMapper**: YES 
  * Needed to get around VT-D if  either unable to disable in BIOS or needed for other operating systems
* **ExternalDiskIcons**: YES 
  * External Icons Patch, for when internal drives are treated as external drives but can also make USB drives internal. For NVMe on Z87 and below you just add built-in property via DeviceProperties.
* **LapicKernelPanic**: NO 
  * Disables kernel panic on AP core lapic interrupt, generally needed for HP systems
* **PanicNoKextDump**: YES 
  * Allows for reading kernel panics logs when kernel panics occurs
* **ThirdPartyTrim**: NO 
  * Enables TRIM, not needed for NVMe but AHCI based drives may require this. Please check under system report to see if your drive supports TRIM
* **XhciPortLimit**: YES 
  * This is actually the 15 port limit patch, don't rely on it as it's not a guaranteed solution for fixing USB. Please create a [USB map](https://usb-map.gitbook.io/project/) when possible as.

The reason being is that UsbInjectAll reimplements builtin macOS functionality without proper current tuning. It is much cleaner to just describe your ports in a single plist-only kext, which will not waste runtime memory and such

### Misc

![Misc](https://i.imgur.com/hotfQPP.png)

**Boot**: Settings for boot screen \(leave as-is unless you know what you're doing\)

* **Timeout**: `5`
  * This sets how long OpenCore will wait until it automatically boots from the default selection
* **ShowPicker**: YES
  * Shows OpenCore's UI, needed for seeing your available drives or set to NO to follow default option
* **UsePicker**: YES
  * Uses OpenCore's default GUI, set to NO if you wish to use a different GUI

**Debug**: Debug has special use cases, leave as-is unless you know what you're doing.

* **DisableWatchDog**: NO \(May need to be set for YES if macOS is stalling on something while booting, generally avoid unless troubleshooting\)

**Security**: Security is pretty self-explanatory.

* **RequireSignature**: NO
  * We won't be dealing vault.plist so we can ignore
* **RequireVault**: NO
  * We won't be dealing vault.plist so we can ignore as well
* **ScanPolicy**: `0` 
* `0` allows you to see all drives available, please refer to OpenCore's DOC for further info on setting up ScanPolicy\(dedicated chapter to come\)

**Tools** Used for running OC debugging tools like clearing NVRAM, we'll be ignoring this

### NVRAM

![NVRAM](https://i.imgur.com/gwsqXCG.png)

**Add**: 4D1EDE05-38C7-4A6A-9CC6-4BCCA8B38C14 \(Booter Path, majogrity can ignore but \)

* **UIScale**:
  * 01: 1080P
  * 02: 2160P\(Enables HIDPI\)

7C436110-AB2A-4BBB-A880-FE41995C9F82 \(System Integrity Protection bitmask\)

* **boot-args**:
  * `-v` - this enables verbose mode, which shows all the behind-the-scenes text that scrolls by as you're booting instead of the Apple logo and progress bar.  It's invaluable to any Hackintosher, as it gives you an inside look at the boot process, and can help you identify issues, problem kexts, etc.
  * `dart=0` - this is just an extra layer of protection against Vt-d issues, keep in mind this requires SIP to be disabled
  * `debug=0x100` - this prevents a reboot on a kernel panic. That way you can \(hopefully\) glean some useful info and follow the breadcrumbs to get past the issues.
  * `keepsyms=1` - this is a companion setting to debug=0x100 that tells the OS to also print the symbols on a kernel panic. That can give some more helpful insight as to what's causing the panic itself.
  * `shikigva=40` - this flag is specific to the iGPU.  It enables a few Shiki settings that do the following \(found [here](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_shiki.hpp#L35-L74)\):
    * 8 - AddExecutableWhitelist - ensures that processes in the whitelist are patched.
    * 32 - ReplaceBoardID - replaces board-id used by AppleGVA by a different board-id. Do note that this generally needed for systems running Nvidia GPUs
* **csr-active-config**: Settings for SIP, generally recommended to manually change this within Recovery partition with `csrutil` via the recovery partition

csr-active-config is set to `E7030000` which effectively disables SIP. You can choose a number of other options to enable/disable sections of SIP. Some common ones are as follows:

* `00000000` - SIP completely enabled
* `30000000` - Allow unsigned kexts and writing to protected fs locations
* `E7030000` - SIP completely disabled
* **nvda\_drv**: &lt;&gt; 
  * For enabling Nvidia WebDrivers, set to 31 if running a [Maxwell or Pascal GPU](https://github.com/khronokernel/Catalina-GPU-Buyers-Guide/blob/master/README.md#Unsupported-nVidia-GPUs). This is the same as setting nvda\_drv=1 but instead we translate it from [text to hex](https://www.browserling.com/tools/hex-to-text)
* **prev-lang:kbd**: &lt;&gt; 
  * Needed for non-latin keyboards

**Block**: Forcibly rewrites NVRAM variables, not needed for us as `sudo nvram` is prefered but useful for those edge cases. Note that `Add` will not overwrite values already present in NVRAM

**LegacyEnable**: NO

* Allows for NVRAM to be stored on nvram.plist, needed for systems without native NVRAM

**LegacySchema**

* Used for assigning NVRAM variables, used with LegacyEnable set to YES

### Platforminfo

![PlatformInfo](https://i.imgur.com/XlxjsAd.png)

For setting up the SMBIOS info, we'll use acidanthera's [_macserial_](https://github.com/acidanthera/MacInfoPkg/releases) application

For this Haswell example, we chose the iMac15,1 SMBIOS. The typical breakdown is as follows:

Haswell with only iGPU - iMac14,1 Haswell with dGPU - iMac14,2 Haswell Refresh - iMac15,1

To get the SMBIOS info generated with macserial, you can run it with the `-a` argument \(which generates serials and board serials for all supported platforms\). You can also parse it with grep to limit your search to one SMBIOS type.

With our iMac15,1 example, we would run macserial like so via the terminal:

`macserial -a | grep -i iMac15,1`

Which would give us output similar to the following:

```text
  iMac15,1 | C02NFZZYFY10 | C02438207QXG2Y7FB
  iMac15,1 | C02P32YJFY10 | C02502303GUG2Y78C
  iMac15,1 | C02P2VZ7FY10 | C02501306QXG2Y7AD
  iMac15,1 | C02NM0EDFY10 | C02444701CDG2Y71H
  iMac15,1 | C02NVHZCFY10 | C02451303CDG2Y7JA
  iMac15,1 | C02QLRZ4FY10 | C02543300GUG2Y7JC
  iMac15,1 | C02QJ0UPFY10 | C02541902GUG2Y7JA
  iMac15,1 | C02QG0NGFY10 | C02539700J9G2Y71M
  iMac15,1 | C02N3XYEFY10 | C02429104J9G2Y7UE
  iMac15,1 | C02QW0M3FY10 | C02552700GUG2Y7JA
```

The order is `Product | Serial | Board Serial (MLB)`

The `iMac15,1` part gets copied to Generic -&gt; SystemProductName.

The `Serial` part gets copied to Generic -&gt; SystemSerialNumber.

The `Board Serial` part gets copied to SMBIOS -&gt; Board Serial Number as well as Generic -&gt; MLB.

We can create an SmUUID by running `uuidgen` in the terminal \(or it's auto-generated via my GenSMBIOS script\) - and that gets copied to Generic -&gt; SystemUUID.

We set Generic -&gt; ROM to either an Apple ROM \(dumped from a real Mac\), your NIC MAC address, or any random MAC address \(could be just 6 random bytes, for this guide we'll use `11223300 0000`\)

**Automatic**: YES

* Generates PlatformInfo based on Generic section instead of DataHub, NVRAM, and SMBIOS sections

**UpdateDataHub**: YES

* Update Data Hub fields

**UpdateNVRAM**: YES

* Update NVRAM fields

**UpdateSMBIOS**: YES

* Updates SMBIOS fields

**UpdateSMBIOSMode**: Create

* Replace the tables with newly allocated EfiReservedMemoryType, use Custom on Dell laptops requiring CustomSMBIOSGuid quirk

### UEFI

![UEFI](https://i.imgur.com/3Cx8kdu.png)

**ConnectDrivers**: YES

* Forces .efi drivers, change to NO will automatically connect added UEFI drivers. This can make booting slightly faster, but not all drivers connect themselves. E.g. certain file system drivers may not load.

**Drivers**: Add your .efi drivers here

**Protocols**:

* **AppleBootPolicy**: NO
  * Ensures APFS compatibility on VMs or legacy Macs, not needed since we're running bare-metal
* **ConsoleControl**: NO
  * Replaces Console Control protocol with a builtin version,  set to YES otherwise you may see text output during booting instead of nice Apple logo. Required for most APTIO firmware
* **DataHub**: NO
  * Reinstalls Data Hub
* **DeviceProperties**: NO
  * Ensures full compatibility on VMs or legacy Macs, not needed since we're running bare-metal

**Quirks**:

* **AvoidHighAlloc**: NO
  * Workaround for when te motherboard can't properly access higher memory in UEFI Boot Services. Avoid unless necessary\(affected models: GA-Z77P-D3 \(rev. 1.1\)\)
* **ExitBootServicesDelay**: `0`
  * Only required for very specific use cases like setting to `5` for ASUS Z87-Pro running FileVault2
* **IgnoreInvalidFlexRatio**: YES
  * Fix for when MSR\_FLEX\_RATIO \(0x194\) can't be disabled in the BIOS, required for all pre-skylake based systems
* **IgnoreTextInGraphics**: NO
  * Fix for UI corruption when both text and graphics outputs happen, set to YES with SanitiseClearScreen also set to YES for pure Apple Logo\(no verbose screen\)
* **ProvideConsoleGop**: YES
  * Enables GOP\(Graphics output Protcol\) which the macOS bootloader requires for console handle
* **ReleaseUsbOwnership**: NO
  * Releases USB controller from firmware driver, avoid unless you know what you're doing
* **RequestBootVarRouting**: NO
  * Redirects AptioMemeoryFix from `EFI_GLOBAL_VARIABLE_GUID` to `OC\_VENDOR\_VARIABLE\_GUID`. Needed for when firmware tries to delete boot entries and is recommended to be enabled on all systems for correct update installation, Startup Disk control panel functioning, etc.
* **SanitiseClearScreen**: NO
  * Fixes High resolutions displays that display OpenCore in 1024x768, required for select AMD GPUs on Z370

## Cleaning up

And now you're ready to save and place it into your EFI

