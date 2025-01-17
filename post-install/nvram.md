# Emulated NVRAM

**Work In Progress**

So this section is for those who don't have native NVRAM, the most common hardware to have incompatible native NVRAM with macOS are the non-Z370 300 series chipsets:

* B360
* B365
* H310
* H370
* Q370
* Z390

## Making nvram.plist

So to make a nvram.plist you'll need 3 things set:

Within your config.plist:

* `DisableVariableWrite`: set to `YES`
* `LegacyEnable`: set to `YES`
* `LegacySchema`: NVRAM variables set\(OpenCore compares these to the variables present in nvram.plist\)
* `ExposeSensitiveData`: set to `0x3`

And within your EFI:

* `FwRuntimeServices.efi` driver\(this is needed for proper sleep, shutdown and other services to work correctly

Now grab the 'LogoutHook.command' and place it somewhere safe like within your user directory:

`/Users/(your username)/LogoutHook/LogoutHook.command`

Open up terminal and run the following:

`sudo defaults write com.apple.loginwindow LogoutHook /Users/(your username)/LogoutHook/LogoutHook.command`

And voila! You have emulated NVRAM, do keep in mind this requires macOS to support the `-x` flag for this to work correctly which is unavailable on 10.12 and below. `nvram.mojave` fixes this by invoking it instead of the system one

