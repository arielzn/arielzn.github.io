---
layout: post
title:  "Installing Arch Linux with encrypted boot and btrfs"
date:   2024-06-03 05:01:01 +0200
categories: archlinux installation encrypted boot btrfs
---

Opinionated guide to install Arch Linux on a device having a fully encrypted
root partition (boot included) using the btrfs filesystem and GRUB bootloader.

## Initial steps

[Download](https://archlinux.org/download/) ISO and sig files and verify the image:

    $ pacman-key -v archlinux-<version>-x86_64.iso.sig

Create a bootable usb:

    # dd bs=4M if=archlinux-*-x86_64.iso of=/dev/sdX status=progress oflag=sync

## Boot live usb and prepare install disk

Disable Secureboot from BIOS. Boot from the usb and get online

    # iwctl station wlan0 connect YOUR-WIFI-SSID

> **TIP**
>
> If you want to install from a graphic environment in order to be able to copy
> paste from e.g. this guide, you could setup a minimal window manager as sway,
> plus a terminal emulator and firefox. As the default ramdisk for the live /
> partition is set to 256M we must [increase it
> first](https://wiki.archlinux.org/title/Archiso#Adjusting_the_size_of_the_root_file_system)
> to have room for the installs.
>
> Steps to get this *spartan* ArchLinux live GUI after booting and
> getting an internet connection
>
>     # mount -o remount,size=4g /run/archiso/cowspace
>     # pacman -Sy
>     # pacman -S sway foot wmenu firefox
>     # sway
>
> Sway shortcuts: `Mod+Enter` open a terminal, `Mod+d` open wmenu to search
> for applications type `firefox` press enter and voilà. `Mod+Shift+E` exit sway.
> By default Mod is the Meta key (the one with ~~Windows~~ Logo).

> **WARNING**
>
> This guide assumes a **full wipe** of your drive to install just Arch Linux
> on it.  If you want to create a dump of your current drive state
>
>     # dd conv=sync,noerror bs=1M if=/dev/nvmeX | pigz -c  > /path/to/backup.img.gz
>
> This image can be [applied back at any
> time](https://wiki.archlinux.org/index.php/Dd#Disk_cloning_and_restore) to
> restore the drive to the dump state.

Cleanup all partitions on the install device

    # wipefs --all /dev/nvmeX

Partition the install device

    # gdisk /dev/nvmeX

Create a partition with 500M for EFI (type code: EF00).
Create a second one using all the remaining space, leave all defaults options.
Write the setup and quit gdisk.
Only in case the drive changes are not captured with `lsblk`, reboot.

Create the filesystem for the EFI partition

    # mkfs.vfat -F32 -n EFI /dev/nvmeXp1

> **NOTE** Since GRUB 2.12rc1 there's only limited support for LUKS2. For the
> moment it can only unlock key slots using the PBKDF2 algorithm, not Argon2.

Setup the main encrypted partition:

    # cryptsetup luksFormat --type luks2 --hash sha256 --pbkdf pbkdf2 --label arch /dev/nvmeXp2
    # cryptsetup luksOpen /dev/nvmeXp2 arch
    # mkfs.btrfs -L btrfs /dev/mapper/arch

Setup btrfs subvolumes

    # mount /dev/mapper/arch /mnt
    # btrfs subvolume create /mnt/root
    # btrfs subvolume create /mnt/home
    # btrfs subvolume create /mnt/varcache
    # btrfs subvolume create /mnt/varlogs
    # btrfs subvolume create /mnt/vartmp
    # umount /mnt

Mount the subvolumes and efi partition

    # mount -o noatime,nodiratime,compress=zstd,subvol=root /dev/mapper/arch /mnt
    # mkdir -p /mnt/{boot/efi,home,var/cache/pacman,var/logs,var/tmp,btrfsroot}
    # mount -o noatime,nodiratime,compress=zstd,subvol=home /dev/mapper/arch /mnt/home
    # mount -o noatime,nodiratime,compress=zstd,subvol=varcache /dev/mapper/arch /mnt/var/cache
    # mount -o noatime,nodiratime,compress=zstd,subvol=varlogs /dev/mapper/arch /mnt/var/logs
    # mount -o noatime,nodiratime,compress=zstd,subvol=vartmp /dev/mapper/arch /mnt/var/tmp
    # mount -o noatime,nodiratime,compress=zstd,subvol=/ /dev/mapper/arch /mnt/btrfsroot
    # mount /dev/nvmeXp1 /mnt/boot/efi

Many guides add also a snapshots subvolume mounted inside `/`, in order to
configure `snapper` to handle automatic snapshots and the possibility to
rollback from the bootloader. I find overkill this setup for my usage, but I
might manually create snapshots of `/` for backup purposes to sync on an
external device. Is good to have the `/var` subdirs above as different
subvolumes to avoid including that on the rootfs snapshot.

## Install Arch

### Initial setup

Install basic packages:

    # pacstrap -K /mnt base linux linux-firmware
                    grub efibootmgr             # boot stuff
                    intel-ucode                 # for intel cpus
                    vim bash-completion         # useful tools

Generate fstab entries

    # genfstab -U /mnt >> /mnt/etc/fstab

Enter the new system

    # arch-chroot /mnt

Set the time zone:

    # ln -sf /usr/share/zoneinfo/Region/City /etc/localtime

Configure hardware clock

    # hwclock --systohc`

Set a hostname

    # echo myhostname > /etc/hostname


Set the root password and add a sudo user

    # passwd
    # useradd -m -G wheel USERNAME
    # passwd USERNAME

Run `visudo` and uncomment the line `%wheel ALL=(ALL) ALL`.

### Personal regional configurations

Configure system locale. Edit `/etc/locale.gen` and enable the desired ones
then

    # locale-gen

Setup for `en_US` as language but `en_GB` specifics for dd/mm/yyyy date, A4
paper and metric system

    # echo LANG=en_US.UTF-8 >> /etc/locale.conf
    # echo LC_TIME=en_GB.UTF-8 >> /etc/locale.conf
    # echo LC_PAPER=en_GB.UTF-8 >> /etc/locale.conf
    # echo LC_MEASUREMENT=en_GB.UTF-8 >> /etc/locale.conf

Setup the vconsole

    # echo KEYMAP=us >> /etc/vconsole.conf
    # echo FONT=eurlatgr >> /etc/vconsole.conf

### Boot setup

Generate initial ramdisk environment. Edit `/etc/mkinitcpio.conf`, for the
`HOOKS` variable add `consolefont` after `keymap` and `encrypt` before
`filesystems`, it should look like this

    HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems fsck)

Build the image

    # mkinitcpio -p linux

Configure GRUB. Edit `/etc/default/grub` and set


    GRUB_CMDLINE_LINUX="cryptdevice=UUID=XXX-XXX-XXX:arch"
    GRUB_ENABLE_CRYPTODISK=y

You can use `blkid` to get the the UUID for the encrypted partition `/dev/nvmeXp2`.

Install GRUB

    # grub-mkconfig -o /boot/grub/grub.cfg
    # grub-install


## Setup a desktop environment and reboot

I'm using Gnome as DE, I prefer to install it before rebooting, so after that
I'll be welcomed by a GUI to start using the system and keep customizing.

    # pacman -S gnome networkmanager
    # systemctl enable NetworkManager
    # systemctl enable gdm

After that exit the chroot and reboot

    # exit
    # umount -R /mnt
    # reboot

You should be greeted by gdm and start enjoying your install.

