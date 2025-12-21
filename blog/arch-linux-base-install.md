---
title: Simple Arch Linux base install
layout: default
---

# Simple Arch Linux base install

## Preface

- This guide aims to create a simple and secure base Arch Linux installation.
- It is not intended for first time users of Arch Linux.

## Goals

- Rootfs encryption
- BRTFS
- UKI
- LTS kernel
- No bootloader

## Prerequisites

- Live environment
- Configured Networking
- SSH (Optional)

## Phases

### Partitioning

Partition scheme

<table>
    <tr>
        <th colspan="2">Block Devce</th>
    </tr>
    <tr>
        <td>EFI - 900M</td>
        <td>Root - Rest of disk</td>
    </tr>
</table>

#### FDisk

After selecting the target disk, open it with fdisk.

```
fdisk /dev/sdX
```

Create a new GPT.

```
Command (m for help): g
```

Create the EFI partition.

```
Command (m for help): n
Partition number (1-128, default 1): 1
First sector (2048-X, default 2048):  
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-X, default X): +900M
Created a new partition 1 of type 'Linux filesystem' and of size 900 MiB.

The signature will be removed by a write command.

Command (m for help): t
Selected partition 1
Partition type or alias (type L to list all): 1
Changed type of partition 'Linux filesystem' to 'EFI System'.
```

Create the root partition.

```
Command (m for help): n
Partition number (2-128, default 2): 2
First sector (1845248-X, default 1845248):  
Last sector, +/-sectors or +/-size{K,M,G,T,P} (1845248-X, default X):  

Created a new partition 2 of type 'Linux filesystem' and of size X GiB.
```

Finally, write the changes to disk.

```
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

#### Formatting

Format EFI as VFAT.

```
mkfs.vfat -F32 -n EFI /dev/sdX1
```

Encrypt root.

```
cryptsetup luksFormat /dev/sdX2  

WARNING!
========
This will overwrite data on /dev/sdX2 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sdX2:  
Verify passphrase:
```

Open cryptroot.

```
cryptsetup open /dev/sdX2 root
Enter passphrase for /dev/sdX2:
```

Format cryptroot as BTRFS.

```
mkfs.btrfs -L ROOT /dev/mapper/root
```

#### BTRFS

Mount BTRFS root.

```
mount -L ROOT /mnt
```

Create subvolumes.

```
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
```

#### Mounting

Mount BTRFS.

```
mount -o compress=zstd,subvol=@ /dev/mapper/root /mnt
mkdir /mnt/home
mount -o compress=zstd,subvol=@home /dev/mapper/root /mnt/home
```

Mount EFI.

```
mkdir /mnt/efi
mount -L EFI /mnt/efi
```

### Install

#### Pacstrap

Install base packages.

```
pacstrap -K /mnt base base-devel linux-lts linux-lts-headers linux-firmware btrfs-progs efibootmgr networkmanager firewalld sudo vim cryptsetup
```

#### Filesystem Table

```
genfstab -U /mnt >> /mnt/etc/fstab
```

#### Chroot

```
arch-chroot /mnt
```

### Configuration

#### System

Set timezone.

```
ln -sf /usr/share/zoneinfo/Your/Location /etc/localtime
hwclock --systohc
```

Set your locale in /etc/locale.gen (Most likely en_US.UTF-8) and run:

```
locale-gen
```

Set LANG in /etc/locale.conf.

```
LANG=en_US.UTF-8
```

Set KEYMAP in /etc/vconsole.conf.

```
KEYMAP=us
```

Set hostname in /etc/hostname.

```
arch
```

#### Users

Add sudoer user.

```
useradd -mG wheel user
```

Set password.

```
passwd user
```

Enable sudoers.

```
EDITOR=vim visudo
```

Uncomment this line.

```
%wheel ALL=(ALL:ALL) ALL
```

#### Services

Enable essential services.

```
systemctl enable firewalld.service systemd-timesyncd.service NetworkManager.service
```

#### Kernel

Find the UUID of the encrypted volume.

```
blkid /dev/sdX2
```

Create the /etc/kernel/cmdline and add:

```
rw loglevel=3 cryptdevice=UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX:root root=/dev/mapper/root rootflags=subvol=@
```

Replace "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX" with the encrypted volume UUID.

#### UKIs

Create paths for UKIs

```
mkdir -p /efi/EFI/Linux
```

Edit /etc/mkinitcpio.conf and change the hooks section to this.

```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
```

Edit /etc/mkinitcpio.d/linux-lts.preset.

```
ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-lts"

PRESETS=('default')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux-lts.img"
default_uki="/efi/EFI/Linux/arch-linux-lts.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
```

Generate UKIs.

```
mkinitcpio -P
```

#### EFI

Configure efibootmgr.

```
efibootmgr --create --disk /dev/sdX --part 1 --label "Linux" --loader '\EFI\Linux\arch-linux-lts.efi' --unicode
```

### Exit and reboot!
