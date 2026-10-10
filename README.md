# OPNsense - Bootloader Upgrade Summarized
OPNsense - Bootloader Upgrade Summarized into a simple HowTo based on the information found in various topics @ OPNsense Forum

During the upgrade of OPNsense from version 26.1.x to 26.7.x there was a need to upgrade the FreeBSD Bootloader because of certains bugs of which the most important one was the fact that the early loading of AMD/Intel Microcode during the boot process caused stalls and various issues.</br>
Later on the developers switched to late loading of the AMD/Intel Microcode and fix the issue.</br>
However seeing the amount of people who did not know what to do and how to do it I think it's good to have all the information from various OPNsense Forum topics about this in a summarized overview instead of having to search for them each time you need to apply the procedure.

A list of commands you might/will need and what they do :
- `sysctl machdep.bootmethod`</br>
This will tell you how your OPNsense Firewall boots : In BIOS or UEFI mode.
- `uname -a`</br>
The well known command to see which Release of FreeBSD and it's Kernel is being used.
- `cat /etc/fstab`</br>
View fstab to see which partitions are mounted on boot and how.
- `gpart show`</br>
Show all your partitions and information about them.
- `gpart show -l`</br>
Another option for the same command which shows the Labels of the partitions.
- `ls -l /boot/efi/efi`</br>
This will show the directories inside your EFI directory located at the /boot partition.</br>
You should see the following two :</br>
<i>/boot/efi/efi/boot</i></br>
and</br>
<i>/boot/efi/efi/freebsd</i></br>
We will take a look at the files located inside them later on!
- `mkdir -p /boot/efi/efi/boot /boot/efi/efi/freebsd` (Usually not needed because the directory already exists!)</br>
- `cp /boot/loader.efi /boot/efi/efi/boot/bootx64.efi`</br>
- `cp /boot/loader.efi /boot/efi/efi/freebsd/loader.efi`</br>
The last two commands are basically the procedure to update the UEFI Bootloader and enough for anyone using a NVMe SSD inside their Firewall.</br>
Reason : You can't boot from NVMe when running in (Legacy) BIOS Mode !!</br>
But let's check some stuff first later on before we do this, OK ?!
- `gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 2 ada0`</br>
This is basically for everyone out there using some sort of SATA storage and booting in Legacy BIOS Mode while using OpenZFS as the filesystem.</br>
It installs the GPT ZFS Bootloader in the MBR of the SATA device.</br>
For the UFS filesystem it would be :</br>
- `gpart bootcode -b /boot/pmbr -p /boot/gptboot -i 2 ada0`</br>
But again : Let's check some stuff first before we do this, OK ?!

So far all the commands you have seen apply to the default OPNsense partitioning schemes for either the UFS or ZFS install method so first we need to find out which method is best for our system before we do anything !!

## So let's begin examining what we need to do and how to do it!

### Step 1 : Checking the Boot Mode : BIOS or UEFI ?

This can be done as following :
```
admin@opnsense:~ # sysctl machdep.bootmethod
machdep.bootmethod: UEFI
```
As you can see in this case we are booting in UEFI Mode.

### Step 2 : Checking our fstab to see if the UEFI Boot partition is mounted during boot ?

This can be done as following :

In case we are dealing with a ZFS File System installation it will look something like this :
```
admin@opnsense:~ # cat /etc/fstab
# Device                Mountpoint      FStype  Options         Dump    Pass#
/dev/gpt/efiboot0       /boot/efi       msdosfs rw              2       2
/dev/ada0p3             none            swap    sw              0       0
```
In case we are dealing with an UFS File System installation it will look something like this :
```
admin@opnsense:~ # cat /etc/fstab
# Device        Mountpoint      FStype  Options Dump    Pass#
/dev/gpt/rootfs /               ufs     rw      1       1
/dev/gpt/efifs  /boot/efi       msdosfs rw      2       2
/dev/gpt/swapfs none            swap    sw      0       0
```
You can see that :</br>
- The UEFI Boot Partition is mounted at /boot/efi</br>
- That it's also mounted in <b>R</b>ead and <b>W</b>rite mode.</br>
That means we don't have to mount it ourselves and we can write the new Bootloader to it!

### Step 3 : Check our partitioning layout and locate the BIOS and UEFI Bootloader Partitions.

This can be done as following :
```
admin@opnsense:~ # gpart show
=>        40  2000409184  ada0  GPT  (954G)
          40      532480     1  efi  (260M)
      532520        1024     2  freebsd-boot  (512K)
      533544         984        - free -  (492K)
      534528    16777216     3  freebsd-swap  (8.0G)
    17311744  1983096832     4  freebsd-zfs  (946G)
  2000408576         648        - free -  (324K)
```
and/or
```
admin@opnsense:~ # gpart show -l
=>        40  2000409184  ada0  GPT  (954G)
          40      532480     1  efiboot0  (260M)
      532520        1024     2  gptboot0  (512K)
      533544         984        - free -  (492K)
      534528    16777216     3  swap0  (8.0G)
    17311744  1983096832     4  zfs0  (946G)
  2000408576         648        - free -  (324K)
```
You can see that :</br>
- The UEFI Boot Partition is the 1st partition with the Partition Label efiboot0</br>
- The BIOS Boot Partition is the 2nd partition with the Partition Label gptboot0</br>
Please note that this is what the default OPNsense partitioning layout looks like when you choose OpenZFS as your File System.

An example of the default OPNsense partitioning layout for UFS as the File System looks like this :</br>
```
admin@opnsense:~ # gpart show
=>        40  1000215136  nda0  GPT  (477G)
          40      532480     1  efi  (260M)
      532520        1024     2  freebsd-boot  (512K)
      533544   982904496     3  freebsd-ufs  (469G)
   983438040    16777136     4  freebsd-swap  (8.0G)
```
and/or
```
admin@opnsense:~ # gpart show -l
=>        40  1000215136  nda0  GPT  (477G)
          40      532480     1  efifs  (260M)
      532520        1024     2  bootfs  (512K)
      533544   982904496     3  rootfs  (469G)
   983438040    16777136     4  swapfs  (8.0G)
```
You can see that :</br>
- The UEFI Boot Partition is the 1st partition with the Partition Label efifs</br>
- The BIOS Boot Partition is the 2nd partition with the Partition Label bootfs</br>

Please also note the difference between the ZFS and UFS examples when it comes to the Storage Device they were installed on :</br>
- `ada0` means we are dealing with a HDD/SSD SATA device.</br>
- `nda0` means we are dealing with a NVMe SSD device.

### Step 4 : Check if all the UEFI Boot related files are there where we expect them to be ?

This can be done as following :
```
admin@opnsense:~ # ls -l /boot/efi/efi
```
Which will show :
```
total 32
drwxr-xr-x  1 root wheel 16384 Jul 14  2024 boot
drwxr-xr-x  1 root wheel 16384 Jul 14  2024 freebsd
```
Now let's check the contents of the <i>boot</i> directory :
```
admin@opnsense:~ # ls -l /boot/efi/efi/boot
total 896
-rwxr-xr-x  1 root wheel 906240 Jul 14  2024 bootx64.efi
```
We got what we need!

The next one is the <i>freebsd</i> directory :
```
admin@opnsense:~ # ls -l /boot/efi/efi/freebsd
```
Which will show :
```
total 896
-rwxr-xr-x  1 root wheel 906240 Jul 14  2024 loader.efi
```
And we got the other one too!

### Step 5 : Let's UPGRADE the UEFI Bootloader first!

We now know that it's safe to update the UEFI Bootloader because :</br>
- The command `sysctl machdep.bootmethod` showed we are booting in UEFI Mode.</br>
- The command `cat /etc/fstab` showed the UEFI Boot Partition is mounted in Read and Write Mode during boot.</br>
- The command `gpart show` or `gpart show -l` showed the UEFI Boot Partition is big enough for the new files when we overwrite the old ones.</br>
Some context : There have been too small partitions in the past!</br>
These days this should not be an issue, but please keep in mind it can happen!</br>
- The commands `ls -l /boot/efi/efi` and `ls -l /boot/efi/efi/boot` and `ls -l /boot/efi/efi/freebsd` showed the files we need to overwrite as there where we need and expect them to be!

Let's UPGRADE the UEFI Bootloader now :
```
admin@opnsense:~ # cp /boot/loader.efi /boot/efi/efi/boot/bootx64.efi
admin@opnsense:~ # cp /boot/loader.efi /boot/efi/efi/freebsd/loader.efi
```
What you have done now is basically overwrite the old UEFI Bootloaders in /boot/efi/efi/ with the new one that's always supplied by a new OPNsense/FreeBSD version in the /boot directory.

In case you :
- Boot in UEFI Mode
- Have no plans to ever boot in BIOS Mode in the future.
- Or are simply using a NVMe SSD which can not boot in BIOS Mode anyway !!</br>
  Storage Device called nda0 in the `gpart` output : Remember ?!

Then the only thing left for you is to REBOOT now and enjoy your new UEFI Bootloader 😬

If not... well... keep reading!

### Step 6 : Let's UPGRADE the BIOS Bootloader too!

So... you want to keep BIOS Boot Mode functionality too or you are simply using it right now and have no other choice BUT upgrading it too ?

No worries! Here we go!

We now know that it's safe to update the BIOS Bootloader because :</br>
- The command `sysctl machdep.bootmethod` showed we are booting in BIOS Mode.</br>
Or you simply want to upgrade it too ofcourse!
- The command `gpart show` or `gpart show -l` showed the BIOS Boot Partition is the 2nd partition with the Partition Label bootfs.</br>
It also showed that in this case we will write the new BIOS Bootloader to the MBR of a device called ada0.

Let's UPGRADE the BIOS Bootloader now :
```
gpart bootcode -b /boot/pmbr -p /boot/gptboot -i 2 ada0
```
Let's break down what the command has done :
- bootcode
  We are 
