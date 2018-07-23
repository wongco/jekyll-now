---
layout: post
title: Dual boot install of Ubuntu 18.04 w/LUKS & Windows Bitlocker (SecureBoot Enabled)
---

Machine: Dell XPS15 9570 (2018 Release)

Working with Ubuntu on a laptop can be a joy considering the maturity of the OS, the software options available and the wide community support it enjoys. However, once you throw in Windows, dual booting and encryption into the mix, the challenges start adding up. Throw in new hardware and you start having to piece together a solution. Here are the issues I ran into and how I was able to resolve them through a combination of testing, searching, referencing other user's ideas and thoughts, and more testing.

## End Goal: Have a fully functioning laptop running Windows 10 w/ Bitlocker & Ubuntu w/ LUKS without Disabling SecureBoot.

I started out preparing the Windows 10 Side first.

## Windows Installation Summary
1. BIOS Config: UEFI Boot Only, TPM On, Secure Boot On, Boot from USB Enabled
2. Installed an HP EX920 M.2 NVME SSD
3. Install Windows Pro 1803 Partitioning only half of the SSD & Enable Bitlocker
4. Once it restarted successfully, I moved onto the Ubuntu portion

## Ubuntu Installation Summary (TLDR Summary)
1. Use a USB2.0 stick/hub to Rufus to write image to usb drive
2. Install Ubuntu Manually referencing instructions from [blog.botux.fr](http://blog.botux.fr/en/2015/09/ubuntu-installation-manual-full-disk-encryption-lvm-on-luks/)
3. Before rebooting, run the command:
> sudo update-pciids 
4. While in the chrooted environment modify /etc/default/grub with the line containing "quiet splash" to:
> GRUB_CMDLINE_LINUX_DEFAULT="i915.alpha_support=1"
5. Restart your computer.
6. Make sure when booting Windows you use the UEFI Loader directly from the BIOS via the F12 Key instead of using GRUB to load Windows in order not to break the boot chain. Otherwise you'll be typing in your Bitlocker Recovery Key a lot.

## Challenges (Details)
* Hardware Support - Out of the of box hardware support
    
    (Intel UHD 630 Graphics iGPU & Nvidia GPU 1050Ti, Killer 1535 Wifi Card, USB3.0 vs USB2.0, Intel 9260NGW Wifi Card)

* Secure Boot - Leaving Secure Boot on which Ubuntu does support (but is often disabled to ease configuration)


---

**USB 3.0 vs USB 2.0** - I started by downloading a copy of the Ubuntu 18.04 64-bit ISO and [Rufus](https://rufus.akeo.ie/) and proceeded to write the iso to a standard SanDisk USB3.0 Thumb Drive to run as a linux liveboot usb stick.  That's when I started getting a few random acpi errors (which were harmless for the most part) and the following message:
> (initramfs) unable to find a live medium containing a live file system

My assumption was that either there was something wrong with my usb drive, or possibly the iso was corrupted or that an error may have occured while writing the iso to the usb drive. After accounting for all the above items including redownloading the iso and rewriting to a different usb drive. I read a few recommendations about some users running into the same issue on other older laptops and their workaround was to use a USB2.0 Hub. Didn't have one of those, but I luckily did have a USB2.0 2GB thumb drive lying around. After writing the iso on Rufus to the USB2.0 the Ubuntu LiveBoot USB stick finally successfully booted.

**Resolution**: Use a USB2.0 Drive or USB2.0 Hub to load up the Ubuntu installer.

---

**Intel UHD 630 Graphics & Nvidia 1050Ti** - Both the integrated GPU and the Nvidia GPU provide different problems.

Nvidia 1050Ti - Two problems with this item
1. On the linux side I wasn't all too concerned about using the discrete chip for Graphics, however the built in driver set kept preventing Ubuntu from loading. I believe this is using the built in nautilus driver set. This forced me to edit /etc/default/grub in recovery mode and change the default option of "quiet splash" to "nomodeset". This disables the GPU drivers and essentially forces the kernal to use the unaccelerated & slow performing basic driver.
2. Trying to install the nvidia drivers from the ubuntu repository (& anywhere else for that matter) nvidia drivers and removing 'nomodeset' would also fail due to SecureBoot not recognizing the drivers.

**Resolution**: Leave the nvidia drivers alone to not break SecureBoot and focus on getting the Intel GPU drivers working.

Intel UHD 630 Graphics - While the 'nomodeset' option allowed for Ubuntu to boot up, the graphics performance was stuck in software mode, resulting in sluggish performance. After testing a few fixes, I ended up with the result of using the built in Intel alpah driver and replaceing 'nomodeset' with

> GRUB_CMDLINE_LINUX_DEFAULT="i915.alpha_support=1"

Ref from: [user1202032](https://askubuntu.com/users/763065/user1202032)<br>
 Answer: https://askubuntu.com/questions/979162/17-10-intel-uhd-630-8700k-unable-to-boot

Once I reboot, was able to confirm graphics accelation was enabled and performance was noticably zippier.

You can check whether driver is running properly via the command:
>lspci -nnk | grep -i vga -A3<br>

Ref from: [L. D. James](https://askubuntu.com/users/29012/l-d-james)<br>
Answer: https://askubuntu.com/a/1027599

Resolution: Change 'nomodeset' in /etc/default/grub to i915.alpha_support=1 to obtain proper accelerated graphics driver support

---

**SecureBoot**

I'm on the side of fence where I'd like to leave SecureBoot enabled unless there is no other alternative. In any case, the issues with SecureBoot were prompted by two separate issues:
1. The Nvidia drivers can't pass through SecureBoot without additional heavy configuration, which I didn't want get into. Since I didn't need the Nvidia GPU in linux, that was fine by me.
2. Intel Wifi 9260NGW - SecureBoot didn't like me trying to update the kernal to a slightly version 4.16.0. The reason for attempting to use a newer kernal is that Ubuntu 18.04 ships with kernal 4.15.0-23 (On my ISO) and contains a buggy Intel Wifi driver that causes the Intel 9260W to run incredibly slowly. (Less than 6Mbps in my experience). Kernal 4.16.0 includes microcode that fixes the issue but unfortunately breaks SecureBoot. As a result I ended up reinstalling the original network card the laptop came with: Killer 1535 Wifi Card which for the most part runs fine in Windows & Ubuntu 18.04.

Resolution: Reinstall original network card

In closing, hopefully some of the items I ran into will help you troubleshoot your own experiences trying to get dual boot + encyrption + secureboot working correctly.