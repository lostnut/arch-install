# archlinux-install

## Resources

Archlinux: https://www.archlinux.org/

Installation guide: https://wiki.archlinux.org/index.php/installation_guide

## Installation
### Download

Download page: https://www.archlinux.org/download/

Download following elements :
- Archlinux ISO:

Choose a mirror and then download ISO: `archlinux-<version>-x86_64.iso` (~600 MB).
  The version number is the release date (e.g. YYYY.MM.DD)
- PGP signature:

Direct download link under checksum section: `archlinux-<version>-x86_64.iso.sig`
The version number is the release date (e.g. YYYY.MM.DD)

### Verify ISO

Verify ISO with `gpg` or `pacman-key`:
- GPG

Documentation: https://wiki.archlinux.org/index.php/GnuPG#Verify_a_signature
```shell
$ gpg --verify archlinux-<version>-x86_64.iso.sig
```
Then verify key on : https://www.archlinux.org/master-keys/

- Pacman-key

```shell
$ pacman-key -v archlinux-<version>-x86_64.iso.sig
```

### Flash media

Documentation: https://wiki.archlinux.org/index.php/USB_flash_installation_media#BIOS_and_UEFI_bootable_USB
- Check for USB drive name :
```shell
$ lsblk
```

- Use dd:
```shell
$ dd bs=4M if=/path/to/archlinux-<version>-x86_64.iso of=/dev/sdx status=progress oflag=sync
```
where `sdx` is the USB drive name found with `lsblk` command.

### Boot from media

This step needs to BOOT on USB drive, depends on motherboard. Have to press <kbd>F1</kbd>,<kbd>F2</kbd> or <kbd>F10</kbd>.

### Set Keyboard Layout

- Check the list of available keymaps
```shell
$ find /usr/share/kbd/keymaps/ -type f
```

- Use loadkeys command:
```shell
loadkeys <keymap>
```

### Verify the boot mode

Verify if the following directory exist:
```shell
$ /sys/firmware/efi/efivars
```

if `efivars` exists, system is booted in UEFI mode, else in BIOS mode (See partitionning)

### Network configuration

- Wired connection:

dhcpcd should connect automatically, see: https://wiki.archlinux.org/index.php/Dhcpcd

- Wireless connection:

use wifi-menu:
```shell
$ wifi-menu
```

- Verify connection:

```shell
$ ping archlinux.org
```

 ### Update the system clock

 ```shell
 $ timedatectl set-ntp true
 $ timedatectl set-timezone <zone>
 ```
 where `zone` can be listed with `timedatectl list-timezones`


 ### Partitionning

 Documentation: https://wiki.archlinux.org/index.php/Partitioning
 Depends on boot mode:


 - for UEFI :
 **WARNING**: EFI System Partition is needed: https://wiki.archlinux.org/index.php/EFI_System_Partition


 Table :

 | Mount point | Partition | Partition type        | Size                   |
 |-------------|-----------|-----------------------|------------------------|
 | /boot       | /dev/sda1 | EFI System Partition  | 550 MB                 |
 | /           | /dev/sda2 | Linux (ext4)          | 40 G                   |
 | [SWAP]      | /dev/sda3 | Linux swap            | 8 G                    |
 | /home       | /dev/sda4 | Linux (ext4)          | Reminder of the device |


 List devices
 ```shell
 $ fdisk -l
 ```

 Launch partitionning utility on device:
  ```shell
 $ fdisk /dev/sdx
 ```
 Then use following commands :
 `m` for help
 `d` for delete previous partitions
 `n` for add partition
 `+X[K,M,G,T,P]` for size

 ### Format partitions

 - ESP (https://wiki.archlinux.org/index.php/EFI_System_Partition)
 ```shell
 $ mkfs.vfat -F32 /dev/sda1
 ```
 - `/` and `/home`
 ```shell
 $ mkfs.ext4 /dev/sdxy
 ```
 - SWAP
 ```shell
 $ mkswap /dev/sdxy
 $ swapon /dev/sdxy
 ```


 ### Mount file systems

 Use command mount and respect the partition table used.

 ```shell
 mkdir <mount-point>
 $ mount /dev/sdxy <mount-point>
 ```

 ### Select the mirrors

- Mirror list:

 Use Archlinux mirror list generator : https://www.archlinux.org/mirrorlist/


 - Reflector:

Documentation : https://wiki.archlinux.org/index.php/reflector

```shell
$ pacman -Sy reflector
$ cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
$ reflector --verbose --latest 5 --sort rate --protocol https --save /etc/pacman.d/mirrorlist
```


 ### Install the base packages

 ```shell
 $ pacstrap /mnt base
 ```

 ### Fstab

Genrate fstab :
 ```shell
 $ genfstab -U /mnt >> /mnt/etc/fstab
 ```
Check fstab:
```shell
$ cat /mnt/etc/fstab
```
### Change root
```shell
$ arch-chroot /mnt
```
### Time zone

```shell
$ ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
$ hwclock --systohc
```

### Locale(s)
Documentation : https://wiki.archlinux.org/index.php/Locale

Uncomment the wanted local in `/etc/locale.gen` :

```shell
$ vim /etc/locale.gen
```

Generate locale(s) :

```shell
$ locale-gen
```
### Hostname

Set hostname:

```shell
$ echo <hostname> > /etc/hostname
$ echo '127.0.1.1 <hostname>.localdomain <hostname>' >> /etc/hosts
```

### Keymap


```shell
$ echo KEYMAP=<keymap> > /etc/vconsole.conf
```

### Language

```shell
$ export LANG=<locale>
```

### Initramfs

```shell
$ mkinitcpio -p linux
```

### Root password

```shell
$ passwd
```

### Boot without boot loader

Documentation : https://wiki.archlinux.org/index.php/EFISTUB

Get UUID :

```shell
$ blkid
```

Set boot with `efibootmgr` :

```shell
$ efibootmgr --disk /dev/sdX --part Y --create --label "Arch Linux" --loader /vmlinuz-linux --unicode 'root=PARTUUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX rw initrd=\initramfs-linux.img' --verbose
```

### Exit and reboot
```shell
$ exit
$ umount -R /mnt
$ reboot
```

Note: Remove the flash drive on reboot.

### Network configuration

Using 'systemd-networkd':
- Documentation :  https://wiki.archlinux.org/index.php/Systemd-networkd

- Install `wpa_supplicant`
```shell
$ pacman -S wpa_supplicant
```
- Enable following unit:
```shell
$ systemctl enable wpa_supplicant@<interface>
$ systemctl enable systemd-networkd.service
$ systemctl enable dhcpcd@<interface>
```
where `interface` can be determined by
```shell
$ ip adress show
```
- Create system networkd configuration
```shell
$ vi /etc/systemd/network/20-wired.network
```

with following content
```
[Match]
Name=<interface>

[Network]
DHCP=yes
```

- Create wpa_supplicant corresponding configuration :
```shell
$ vi /etc/wpa_supplicant/wpa_supplicant-<interface>.conf
```
with following content
```
ctrl_interface=/run/wpa_supplicant
ctrl_interface_group=wheel
network={
  password="password"
  ssid="SSID"
}
```
### Users
```shell
$ useradd -m -g users -G "<comma-separated-list-of-additional-groups>" <username>
$ passwd <username>
```

### Sudo
```shell
$ pacman -S sudo
```

Add sudo privileges to wheel group :
```shell
$ visudo
```

And uncomment line: `%wheel ALL=(ALL) ALL`

Can verify privileges with: 

```shell
$ sudo -lU <username>
```

### gpg
Documentation : https://wiki.archlinux.org/index.php/GnuPG
Generate a gpg key
```shell
$ gpg --full-key-gen
```

### Passwords

Documentation : https://wiki.archlinux.org/index.php/Pass

```shell
$ pass init <gpg-id>
```

Then inserts all passwords : 
```shell
$ pass insert password/full/description
```

### Ssh
```shell
$ pacman -S openssh
$ vi /etc/ssh/sshd_config
```
And uncomment line : `PermitRootLogin` and set it to `no`.

### Graphic driver
```shell
$ pacman -S xf86-video-nouveau
```

### Xorg
```shell
$ pacman -S xorg-server xorg-xinit
```

### i3
```shell 
$ pacman -S i3-gaps
```

### Xinit
Add an xinit file: 

```shell
$ vi ~/.xinitrc
```
with the following content :
```
exec i3
```

## Applications

- borg
- fd
- i3-gaps
- openssh
- pass
- rxvt-unicode
- sudo
- ttf-hack
- vim
- xorg-server
- xorg-xinit
- zsh

