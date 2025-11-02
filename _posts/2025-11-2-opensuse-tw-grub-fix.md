---
layout: post
title: "Rescue OpenSUSE Tumbleweed (recreate Grub config from Rescue System)"
tags: [personal]
comments: false
---

In last 6 months time, twice I had to get my work machine's system board (aka the motherboard) replaced.  
First for an "Integrated Graphics Error" (one day I got these very annoying beeps on my work machine, I ran a Lenovo Smartbeep scan using their mobile app, and it suggested to contact (immediately) Lenovo support and request to change the system board).  
Second time, the Wi-Fi (infact all wireless) stopped working on my machine. 

For a few weeks following the first system board replacement, I thought it was some wifi firmware mismatch issue on my OpenSUSE Tumbleweed (TW) machine, because its a rolling release, once in a while distribution upgrade breaks stuff, so its a normal thing.  
But I remember for the first few weeks after the hardware replacement - it worked sometimes, it will detect Wi-fi but then other times, it will drop wifi entirely. And then from last ~1.5 months - I have been relying entirely on an Ethernet for Internet on my work machine.  
And that won't work when I'm travelling.  
So, I tried booting with a live USB stick into a Mint Cinnamon machine, and it was clear - it's not just a TW issue, Mint also didn't detect any wireless network. Zero, nil, nothing.    
(Not to say, over last months, when I thought it was some firmware issue, I had tried many things, lots around the `iwlwifi` firmware, but nothing worked. I have been eying many many upstream kernel bugzillas related to iwlwifi and I was convinced it was a firmwae issue :/).  

Now, very fortunately, Lenovo Premium Support just works (for me it did! Twice! I contacted them twice in last 6 months, and both times an engineer visited almost on the next day or in two.)  
Both times, they replaced the mother board (my work machine is a ThinkPad Workstation and every thing is just stuck on the mother board, so any chip dies and it requires a full system board replacement).

Both times when the mother board is replaced, it's almost a new machine, only with the same old storage.  
(very very important storage. Because it still contains my old TW OS partitions and data, and all the precious system configurations which takes a very long to configure again).  
I did ran backups before both replacements, but still it's a pain if I have to do a fresh OS reinstallation and setup everything again, in the middle of a work week.

So, when the system board is replaced, I think it refreshes the BIOS and stuff, and my grub menu no longer sees the TW OS partitions and so it just directly boots into the mighty Windows Blue Screen screaming the system can't be fixed, and I need to do a fresh install.

But don't get fooled by that (not immediately, check once).  
Chances are that the old OS partitions are still there, just not being detected by the Grub bootloader.  
And that was the case for me (both times).

And not to my surprise, the OpenSUSE TW "Rescue System" menu came to my resuce!  
(well, to my surprise, because let's not forget TW is a rolling release OS. So things _can_ go south very very quickly.) 

I did the following:

- I created a live USB stick with OpenSUSE Tumbleweed.  
  (It helped to have a stick with a full Offline image, and not the tiny Network image which will pull every single thing from Internet.  
  Because remember "the Wi-FI" not working on my machine.  
  Well I could have connected to Ethernet but still, the lesson is to have a stick ready with an offline image so it should just boot.)
  
- Now, put it in the machine, go to "Boot Menu" (F10, IIRC), and pick the option to boot from the Live USB stick.  
  It will go to a grub menu.  
  Skip all the immediate "OpenSUSE Tumbleweed installation, etc" menu options.
  Go to "More ..." and then "Rescue System".  
  It will do the usual "loading basic drivers > hardware detection > ask to pick a keyboard layout, et. al" and then give me the "Resuce Login:" prompt.  
  Remember the username is "root" and there is no password.  
  With that, I enter `tty1:resuce:/ #`.

  Now run the following set of commands:
  
  ```
  # first things first, check my disk and partitions (if they still exists, I move on. Otherwise, all is gone and nothing more to do!)

  fdisk -l
  ## gives me something like following (truncated ofcourse, to the important bits)
  Disk /dev/nvme0n1: xx.xx GiB, xxxxxx bytes, xxxxxx sectors
  Disk model: xxx PC xxxx xxxxx-xxxx-xxxx          
  Units: sectors of 1 * 512 = 512 bytes
  Sector size (logical/physical): 512 bytes / 512 bytes
  I/O size (minimum/optimal): 512 bytes / 512 bytes
  Disklabel type: xxx
  Disk identifier: xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxxxx

  Device           Start   End     Sectors   Size       Type
  /dev/nvme0n1p1   xxx     xxxx    xxxxx     260M       EFI System
  /dev/nvme0n1p2   xxxx    xxxxx   xxxx      xxx.xxG    Linux filesystem
  /dev/nvme0n1p3   xxxxx   xxxxx   xxxx      2G Linux   swap

  # in my case, the disk that has the OpenSUSE tumbleweed is "/dev/nvme0n1".
  # "/dev/nvme0n1p1" is the EFI partition
  # "/dev/nvme0n1p2" is the root partition
  # "/dev/nvme0n1p3" is the Swap partition
  
  # From this step onwards:
  # I need "/dev/nvme0n1p1" (EFI System) and "/dev/nvme0n1p2" (Linux Filesystem)
  
  # I need to mount these two partitions under "/mnt"
  # (make sure the `/mnt` directory is empty before mounting anything to it)

  cd /mnt
  ls  # should be empty
  cd ..

  mount /dev/nvme0n1p2 /mnt
  mount /dev/nvme0n1p1 /mnt/boot/efi
  
  # next I need to mount "/dev", "/proc", "/sys", "/run" from the live environment into the mount directory

  mount -B /dev /mnt/dev
  mount -B /proc /mnt/proc
  mount -B /sys /mnt/sys
  mount -B /run /mnt/run

  # now chroot into the "/mnt" directory
  # the prompt will turn into `resuce:/ #` from the earlier `tty1:resuce:/ #`

  chroot /mnt   

  # now make the EFI variables available

  mount -t efivarfs none /sys/firmware/efi/efivars

  # now reinstall grub2, with `grub2-install`

  grub2-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=opensuse
  ## should output something like:
  Installing for x86_64-efi platform.
  Installation finished. No error reported.

  # then probe for other operating systems on this machine
  
  os-prober
  ## should output something like (and because my machine originally came with Windows, it still shows remnants of that)
  /dev/nvme0n1p1@/EFI/Microsoft/Boot/bootmgfw.efi:Windows Boot Manager:Windows:efi

  # now, create a new grub configuration file using grub2-mkconfig

  grub2-mkconfig -o /boot/grub2/grub.cfg
  ## should output something like:
  Generating grub configuraiton file ...
  Found theme: /boot/grub2/themes/opneSUSE/theme.txt
  Found linux image: /boot/vmlinuz-x.xx.x-x-default
  Found initrd image: /boot/initrd-x.xx.x-x-default
  Warning: os-prober will be executed to detect other bootable partitions.
  Its output will be used to detect bootable binaries on them and create new boot entries.
  Found Windows Boot manager on /dev/nvme0n1p1@/EFI/Microsoft/Boot/bootmgfw.efi
  Adding boot menu entry for UEFI Firmware Settings ...
  done

  # If all good so far, exit out of chroot

  exit

  # Reboot the machine.
  # And remove the installation media

  reboot
  ```

- Once the system is rebooted (remember I have removed the installation media at this point), I go to the BIOS to confirm the boot order.

- Now, restart the machine and it should have a grub menu with proper "OpenSUSE Tumbleweed" boot entry.  
  And it should just boot.  
  Boot and login! It should work now! (It did, twice I followed this process and it did).  
  (Also, a note, when I reboot into OpenSUSE tumbleweed after this new grub config creation on a new system board, I need to also make sure "Secure Boot" is disabled in BIOS menu.  
  Otherwise it will not allow the OpenSUSE TW to boot.  
  It didn't for me because my Secure Boot was enabled.  
  So, I had to disable it. And then it worked.  
  After first successful boot, I think I can enable it again.)

None of this process is my own making.  
The whole credit goes to this very useful demo - [KMDTech: How To Fix Grub in OpenSUSE | UEFI](https://youtu.be/6L3i4Zgb8Y8?si=D96jkLbSR1Dzr416). (Thank you!)  
It worked twice for me without a single hiccup.  
(but if you need to follow, please do it at your own discretion ofcourse, after checking, cross-checking everything you're typing).
  



  
  
