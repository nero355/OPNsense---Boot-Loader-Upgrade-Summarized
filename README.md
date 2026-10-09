# OPNsense - Bootloader Upgrade Summarized
OPNsense - Bootloader Upgrade Summarized into a simple HowTo based on the information found in various topics @ OPNsense Forum

During the upgrade of OPNsense from version 26.1.x to 26.7.x there was a need to upgrade the FreeBSD Bootloader because of certains bugs of which the most important one was the fact that the early loading of AMD/Intel Microcode during the boot process caused stalls and various issues.
Later on the developers switched to late loading of the AMD/Intel Microcode and fix the issue.
However seeing the amount of people who did not know what to do and how to do it I think it's good to have various OPnsense Forum topics about this in a summarized overview instead of having to search for them each time you need to apply the procedure.

A list of commands you might/will need and what they do :
- sysctl machdep.bootmethod</br>
This will tell you how your OPNsense Firewall boots : In BIOS or UEFI mode.
- uname -a</br>
The well known command to see which Release of FreeBSD and it's Kernel is being used.
- cat /etc/fstab</br>
View fstab to see which partitions are mounted on boot and how.
- gpart show</br>
Show all your partitions and information about them.
- gpart show -l</br>
Another option for the same command which shows the Labels of the partitions.
- ls -l /boot/efi/efi</br>
This will show the directories inside your EFI directory located at the /boot partition.
You should see the following two :
/boot/efi/efi/boot
and
/boot/efi/efi/freebsd
We will take a look at the files located inside them later on!
- mkdir -p /boot/efi/efi/boot /boot/efi/efi/freebsd (Usually not needed because the directory already exists!)</br>
- cp /boot/loader.efi /boot/efi/efi/boot/bootx64.efi</br>
- cp /boot/loader.efi /boot/efi/efi/freebsd/loader.efi</br>
The last two commands are basically the procedure to update the UEFI Bootloader and enough for anyone using a NVMe SSD inside their Firewall.
Reason : You can't boot from NVMe when running in (Legacy) BIOS Mode !!
But let's check some stuff first later on before we do this, OK ?!
- gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 2 ada0</br>
This is basically for everyone out there using some sort of SATA storage and booting in Legacy BIOS Mode while using OpenZFS as the filesystem.
It installs the ZFS Bootloader in the MBR/GPT Partition of the SATA device.
But again : Let's check some stuff first before we do this, OK ?!

So far all the commands you have seen apply to the default OPNsense partitioning schemes for either the UFS or ZFS install method so first we need to find out which method is best for our system before we do anything !!
