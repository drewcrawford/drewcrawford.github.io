---
layout: page
title: install windows 8.1 to external disk
---

Here's how to install Windows 8 to an external disk on a Mac.

Note that, for reasons that are a bit of a mystery to me, you can't get the install to work with a USB 3 external HD.  However, it's completely fine to install to a USB 2 HD and then image the drive over to a USB3 one.

1.  Install [Virtualbox](www.virtualbox.org)
2.  Install VirtualBox OracleVM Extension Pack.  It's a [separate download](https://www.virtualbox.org/wiki/Downloads).
2.  Pull down the Windows 8.1 VM for VirtualBox from [modern.ie](https://www.modern.ie/en-us/virtualization-tools#downloads)
3.  Extract with [The Unarchiver](https://itunes.apple.com/us/app/the-unarchiver/id425424353?mt=12)
4.  Import image to VirtualBox
5.  Connect USB HD to Mac
6.  In Disk Utility, partition the USB HD with 1 Partition of "Free Space".
6.  Unmount USB HD using Disk Utility
6.  Under "ports->USB" add the HD to the VM
6.  Enable USB2 in "ports->USB"
7.  Launch VM
9.  Open compmgmt.msc by typing it into Explorer
10. Down in Storage Manager, format the unallocated space as an NTFS partition
8.  Install [WinToUsb](http://www.easyuefi.com/wintousb/) on VM
9.  By hook or by crook, get your Windows 8 ISO inside the virtual machine.  In my case I did it over SMB from a fileserver, YMMV.
10. Run through the WinToUsb setup, choosing the 200MB parition as the EFI partition, and the large NTFS partition as the system partition.
11. Go brew some coffee, take a walk, etc.
12. Turn off the Mac
13. Hold down Option on boot, choose EFI boot
14. Now you're running Windows.

If you want to copy over to a USB3 drive:

1.  Launch Disk Utility
2.  Partition your USB3 drive with 1 MS-DOS partition.  Make sure the parition table is GUID, not MBR.  MBR won't boot.
3.  Use [WinClone](http://www.twocanoes.com/winclone/) to image the old drive.  It's $30 at the time of this writing, but worth it if you're doing serious Windows work.  The image that comes out the other side is tiny, <4GB in my case.
4.  Restore the image onto the new HD.