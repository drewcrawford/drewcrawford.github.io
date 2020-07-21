---
layout: page
title: Installing Big Sur beta on an external disk
---

Here's how I installed Big Sur beta on an external disk.

1.  For macs that have a T2, you first need to boot into recovery mode and use the [Startup Security Utility](https://support.apple.com/en-us/HT208198).  Hold Cmd-R at boot, choose Utilities -> Startup Security Utility, and choose the "Allow booting from external media" setting.  Note that you should probably remember to re-enable this when you retire your external disk workflow.
2.  Erase the entire disk.  Open Disk Utility, in the "View" sidebar button, choose "Show All Devices" (not "Show only Volumes").  By the way, this setting fixes one of my top annoyances with Disk Utility.  Select the parent disk, choose Erase.  Choose APFS as the partition, or else you will get weird errors later about "Unable to verify startup disk" (FB8092255)  Do make sure the scheme is "GUID Partition Map".
3.  Download the profile from developer.apple.com.  This should open system preferences with a new "software update" to Big Sur.
4.  Run through the process in System Preferences.  Note that it won't actually update your mac, it will download and launch the Big Sur installer in a separate window.
5.  In the Big Sur installer, choose the external disk you formatted in 2.
6.  Reboot.  Install should complete.