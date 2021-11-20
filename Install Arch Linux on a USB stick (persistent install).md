## 1. Connect to the Internet (Wireless)

Boot Arch Linux live media and then enter iwd mode:
```
iwctl
```

Search for the Wifi adapter (after the command above)
```
device list
```

Search for Wifi networks available using the interface found using the command above
```
station wlan0 scan
```

To see the list of the Wifi networks
```
station wlan0 get-networks
```

Connect to a Wifi network (this command can be executed right after entering IWD mode. Tab completion is possible for SSID)
```
station wlan0 connect "SSID"
```

Exit iwc mode and check if wla0 received an IP address and if Internet is working.
```
ip a

ping -c 5 archlinux.org
```


## 2. Update the system clock

Turn network time synchronization On
```
timedatectl set-ntp true
```
To check status
```
timedatectl status
```

### IF REQUIRED...
To list all timezones
```
timedatectl list-timezones | grep Sydney
```
To set a timezone
```
timedatectl set-timezone Australia/Sydney
```


## 3. Partition and Formatting
List all devices to make sure which one is the target USB drive: dev/sdX
```
lsblk
```
**NOTE:** it there are many disk, make sure to select the right one. Multiple disks will
show like /dev/sda or /dev/sdb, or /dev/sdb
```
cfdisk /dev/sdX # Select GTP if it asks to select a label type
```

Create an EFI and Linux partition:
- EFI: 500MB
- Linux: remaining of the disk

**NOTE:** create a BIOS partition of 10MB if required to boot the USB on old machines (no need to format this partition). However, I only used the EFI one.

View the new block layout of the target USB device:
```
lsblk /dev/sdX
```

### EXAMPLE
```
NAME     MAJ:MIN RM  SIZE  RO  TYPE MOUNTPOINTS
sdX        8:16  1   59.8G  0  disk
  ├─sdX1   8:17  1     10M  0  part # BIOS
  ├─sdX2   8:17  1    477M  0  part # EFI
  └─sdX3   8:19  1   59.3G  0  part # Linux
```

Format the 500MB EFI partition with a FAT32 filesystem
```
mkfs.fat -F32 /dev/sdX2
```

Format the Linux partition with an ext4 filesystem. The -O "^has_journal" is to disable journaling on the partition, required to USB installs.
```
mkfs.ext4 -O "^has_journal" /dev/sdX3
```

## 4. Installing the Base packages and some configurations
### Mounting the partitions
Mount the ext4 formatted partition as the root filesystem
```
mount /dev/sdX3 /mnt
```

Mount the FAT32 formatted EFI partition to boot:
```
mkdir -p /mnt/boot/EFI
mount /dev/sdX2 /mnt/boot/EFI
```

### Install the base system 
Vim is required to edit some files along the way
```
pacstrap -i /mnt base base-devel linux linux-firmware vim
```

### Generate fstab
Generate a new fstab using UUIDs as source identifiers
```
genfstab -U /mnt >> /mnt/etc/fstab
```

Check the generated /etc/fstab:
```
cat /mnt/etc/fstab
```

### Configuring Arch
Change root into the new system:
```
arch-chroot /mnt
```

Set the timezone. Use tab for auto-completion:
```
ln -sf /usr/share/zoneinfo/Australia/Sydney /etc/localtime
```

Run hwclock to generate /etc/adjtime:
```
hwclock --systohc
```

EDIT /etc/locale.gen and uncomment en_US.UTF-8 UTF-8. Generate the locales by running:
```
vim /etc/locale.gen # Make appropriate change then save and exit
locale-gen
```

Set default locale:
```
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Set hostname
```
echo "arch" > /etc/hostname
```

Edit /etc/hosts with VIM and add the following lines
```
127.0.0.1    localhost
::1          localhost
127.0.1.1    arch.localdomain    arch
```
Save and exit VIM

**NOTE:** # To prevent excessive writes to the USB and improve system performance, use the noatime mount option in fstab. Open /etc/fstab in an editor:
```
vim /etc/fstab
```

Change each relatime or atime option to noatime.
```
===================================================
	# /dev/sdi3
  UUID=uuid1  /      ext4  rw,noatime 0 1

  # /dev/sdi2
  UUID=uuid2  /boot/EFI  vfat  rw,noatime,options 0 2
===================================================
```

Configure journaling - To prevent even further excess file writing.
```
mkdir -p /etc/systemd/journald.conf.d
vim /etc/systemd/journald.conf.d/usbstick.conf
```

Edit the file to contain:
```
[Journal]
Storage=volatile
RuntimeMaxUse=30M
```
Save and exit VIM

Install battery support
```
pacman -S acpi
```

### Configure HOOKS order
EDIT the file below and look for HOOKS. Add the words **block** and **keyboard** _before_ **autodetect**. 
Remove block and keyboard words from the end of the line.
```
vim /etc/mkinitcpio.conf
mkinitcpio -p linux
```

### Installing and configuring the Bootloader
Configure bootloader: EFI and MBR/BIOS
Install the grub and efibootmgr packages and other extra packages:
```
pacman -S grub efibootmgr dosfstools os-prober mtools
```

View the current block devices to determine the target USB device again if requried:
```
lsblk
```

Setup GRUB for MBR/BIOS booting mode:
```
Have not tested this.
```

Setup GRUB for UEFI booting mode:
```
grub-install --target=x86_64-efi --efi-directory=/boot/EFI --bootloader-id=ARCH --removable --recheck
```

Generate a GRUB configuration:
```
grub-mkconfig -o /boot/grub/grub.cfg
```

### Networking
Enable NetworkManager at boot
```
pacman -S iwd dhcpcd networkmanager dialog
systemctl enable iwd dhcpcd NetworkManager
```

### Setting root password and creating a new user
Set root password
```
passwd
```

Create a new user
```
useradd -m -g users -G wheel <username>
```

Set user password
```
passwd <username>
```

Install sudo package
```
pacman -S sudo
```

To enable sudo access to the group WHEEL edit /etc/sudoers and uncomment:
```
%wheel ALL=(ALL) ALL
```

### Exit chroot, umount and reboot
Logout of the chroot:
```
exit
```

Unmount the USB:
```
umount /mnt/boot/EFI /mnt && sync
reboot
```



## POST installtion
Connect to the Internet.

### Installing Microcode
For Intel CPU
```
pacman -S intel-ucode
```

For AMD CPU
```
pacman -S amd-ucode
```

Rename grub.cfg to grub.cfg.bkp and regenerate grub.cfg file again
```
mv /boot/grub/grub.cfg /boot/grub/grub.cfg.bkp
grub-mkconfig -o /boot/grub/grub.cfg
```

### Enable multilib repository
Uncomment the line below in /etc/pacman.config to run 32-bit application on a 64-bit systems
```
[multilib]
include = /etc/pacman.d/mirrorlis
```
Run pacman to update mirrorlist
```
pacman -Syu
```


# Troubleshooting
### Slow download
Update mirrorlist with the best available/fastest mirrors. It's a good idea to do a backup of the mirrorlist file before saving the results of the reflector command to mirrorlist.
```
reflector -c <country> --sort rate --save /etc/pacman.d/mirrorlist
pacman -Syy
```
