## Installation on Dell XPS

Please also consult official documentation: https://wiki.archlinux.org/index.php/Installation_Guide

### Enter BIOS with F2 and configure:

- "System Configuration" > "SATA Operation": "AHCI"
- "Secure Boot" > "Secure Boot Enable": "Disabled"

### Boot from USB
**Make sure to boot from USB using UEFI!**

### Set desired keymap
`loadkeys en_US-utf8`

### (Optional) Connect to wifi
`wifi-menu`

Once done it can take some seconds - confirm it worked using ping

`ping 8.8.8.8`

### Sync clock
`timedatectl set-ntp true`

### Create two partitions:
- 1 1000MB EFI partition # Hex code ef00
- 2 100% Linux partiton (to be encrypted) # Hex code 8300

**CAUTION** Find the correct disk/partition names for yourself using `lsblk`. From here on I am using mine as an example. Do not blindly copy paste these, it might not work or you might destroy partitions you don't want to destroy.

`cgdisk /dev/nvme0n1`

### (Optional) Install f2fs-tools
`pacman -S f2fs-tools`

### Formatting and encyption
**Boot** partition

`mkfs.fat -F32 /dev/nvme0n1p1`

**Root** partition

`cryptsetup luksFormat --type=luks2 /dev/nvme0n1p2`

`cryptsetup open /dev/nvme0n1p2 luks`

(choose filesystem - assuming f2fs)

`mkfs.f2fs -l luks /dev/mapper/luks`

### Mount root and boot partition
`mount /dev/mapper/luks /mnt`

`mount /dev/nvme0n1p1 /mnt/boot`

### Change pacman mirror priority, move closer mirror to the top
`vim /etc/pacman.d/mirrorlist`

### Install the base system plus a few packages (f2fs-tools if you chose f2fs as your partition)
`pacstrap /mnt base linux linux-firmware zsh vim git sudo efibootmgr wpa_supplicant dialog iw f2fs-tools`

### Generate fstab
`genfstab -L /mnt >> /mnt/etc/fstab`

### Enter the new system
`arch-chroot /mnt`

### Setup time
`rm /etc/localtime`

`ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime`

`hwclock --systohc`

### Generate required locales
`vim /etc/locale.gen`

`locale-gen`

### Set desired locale
`echo 'LANG=en_US.UTF-8' > /etc/locale.conf`

### Set desired keymap and font
`echo 'KEYMAP=us' > /etc/vconsole.conf`

### Set the hostname
`echo '<hostname>' > /etc/hostname`

### Add hostname to /etc/hosts:
`vim /etc/hosts`

```
127.0.0.1	localhost
::1		localhost
127.0.1.1	<hostname>.localdomain <hostname>
```

### Set password for root
`passwd`

### Add real user
`useradd -m -g users -G wheel -s /bin/zsh <username>`

`passwd <username>`

`echo '<username> ALL=(ALL) ALL' > /etc/sudoers.d/<username>`

### Configure mkinitcpio with modules needed for the initrd image
`vim /etc/mkinitcpio.conf`

```
# (Optional) For f2fs required - add module crypto-crc32
MODULES=(crypto-crc32)

# Important - Use the correct order
HOOKS=(base systemd autodetect modconf block keyboard sd-vconsole sd-encrypt filesystems)
```

### Regenerate initrd image
`mkinitcpio -p linux`

### Setup systemd-boot
`bootctl --path=/boot install`

### Enable Intel microcode updates
`pacman -S intel-ucode`

### Create bootloader entry
Get luks-uuid with: 

`cryptsetup luksUUID /dev/nvme0n1p2`

Create the entry:

`vim /boot/loader/entries/arch.conf`
```
title		Arch Linux
linux		/vmlinuz-linux
initrd		/intel-ucode.img
initrd		/initramfs-linux.img
options		rw luks.uuid=<uuid> luks.name=<uuid>=luks root=/dev/mapper/luks
```

### Set default bootloader entry
`vim /boot/loader/loader.conf`
```
default		arch
```


### (Optional) - Setup Gnome
`pacman -S gnome`

`systemctl enable gdm`

### Exit, unmount and reboot
`exit`

`umount -R /mnt`

`reboot`

