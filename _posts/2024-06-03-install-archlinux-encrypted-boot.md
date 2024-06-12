---
layout: post
title:  "Installing Arch Linux with encrypted boot and btrfs"
date:   2024-06-03 01:01:01 +0200
categories: archlinux installation encrypted boot btrfs
---

Opinionated guide to install Arch Linux on a device having a fully encrypted
root partition (boot included) using the btrfs filesystem and GRUB bootloader.

### Useful trick

{% alert tip %}
This is a tip!
{% endalert %}


If you want to install from a graphic environment after booting the official
Arch Linux install media, you can install a minimal window manager as sway,
plus a terminal emulator and firefox.

This should allow to follow this guide from the target installation machine and
copy paste the commands.

As the default ramdisk for the cowspace is set to 256M we must [increase it
first](https://wiki.archlinux.org/title/Archiso#Adjusting_the_size_of_the_root_file_system)
to have room for the installs.

Steps to get this *spartan* official ArchLinux live GUI after booting and
getting an internet connection:

    # mount -o remount,size=4g /run/archiso/cowspace
    # pacman -Sy
    # pacman -S sway foot wmenu firefox
    # sway

If you go for this you should be comfortable with a tiling window manager.
Shortcut tips: `Mod+Enter` will open a terminal, `Mod+d` open wmenu to search
for applications type `firefox` press enter and voilà. `Mod+Shift+E` exit sway.
By default Mod is the Meta key (the one with ~~Windows~~ Logo).


## Initial steps

1. Disable Secureboot from BIOS.
1. [Download](https://archlinux.org/download/) ISO and sig files
1. Verify the image: `$ pacman-key -v archlinux-<version>-x86_64.iso.sig`
1. Create a bootable usb with:

        # dd bs=4M if=archlinux-*-x86_64.iso of=/dev/sdX status=progress oflag=sync

## Boot usb and prepare install disk

**WARNING**: This guide assumes a **full wipe** of your drive to install just Arch Linux on it.
If you want to create a dump of your current drive state:

    # dd conv=sync,noerror bs=1M if=/dev/nvmeX  of=/mnt/extdrive/backup_nmveX.img status=progress

This image can be [applied back at any
time](https://wiki.archlinux.org/index.php/Dd#Disk_cloning_and_restore) to the
nvme drive to restore it completely to the dump state.

Get online `# iwctl station wlan0 connect YOUR-WIFI-SSID`
Cleanup all partitions on the install device

    # wipefs --all /dev/nvmeX

Partition the install device:

    # gdisk /dev/nvmeX

Create a partition with 500M for EFI (EF00).
Create another using the remaining space, leave all defaults options.
Only in case the drive changes are not captured with `# lsblk` reboot.

Create the filesystem for the EFI partition

    # mkfs.vfat -F32 -n EFI /dev/nvmeXp1

Setup the main encrypted partition:

    # cryptsetup luksFormat --type luks2 --hash sha256 --pbkdf pbkdf2 --label arch /dev/nvmeXp2
    # cryptsetup luksOpen /dev/nvmeXp3 arch
    # mkfs.btrfs -L btrfs /dev/mapper/arch

Setup btrfs subvolumes

    # mount /dev/mapper/arch /mnt
    # btrfs subvolume create /mnt/root
    # btrfs subvolume create /mnt/home
    # btrfs subvolume create /mnt/pkgs
    # btrfs subvolume create /mnt/varlogs
    # btrfs subvolume create /mnt/vartemp
    # umount /mnt

Mount the subvolumes and efi

    # mount -o noatime,nodiratime,compress=zstd,subvol=root /dev/mapper/arch /mnt
    # mkdir -p /mnt/{boot/efi,home,var/cache/pacman,var/logs,var/tmp,btrfsroot}
    # mount -o noatime,nodiratime,compress=zstd,subvol=home /dev/mapper/arch /mnt/home
    # mount -o noatime,nodiratime,compress=zstd,subvol=pkgs /dev/mapper/arch /mnt/var/cache/pacman
    # mount -o noatime,nodiratime,compress=zstd,subvol=varlogs /dev/mapper/arch /mnt/var/logs
    # mount -o noatime,nodiratime,compress=zstd,subvol=vartemp /dev/mapper/arch /mnt/var/tmp
    # mount -o noatime,nodiratime,compress=zstd,subvol=/ /dev/mapper/arch /mnt/btrfsroot
    # mount /dev/nvmeXp1 /mnt/boot/efi

Many guides add also a snapshots subvolume mounted inside `/`, in order to
configure `snapper` to handle automatic snapshots and the possibility to
rollback. I'm not planning to have that setup, but if I want to manually
create snapshots of `/` for backup purposes to sync on an external device, is
good to have `/var/logs` and `/var/tmp` as different subvolumes to avoid
including that on the backup.


## Install Arch

1. Install basic packages:

        # pacstrap /mnt base base-devel
                        grub efibootmgr             # boot stuff
                        intel-ucode                 # for intel cpus
                        vim git openssh ansible     # tools to deploy playbooks

1. Generate fstab entries: `# genfstab -U /mnt >> /mnt/etc/fstab`
1. Enter the new system: `# arch-chroot /mnt`
1. Set the time zone: `# ln -sf /usr/share/zoneinfo/Region/City /etc/localtime`
1. Configure hardware clock: `# hwclock --systohc`
1. Configure system locale:
  * Edit `# vim /etc/locale.gen` and enable the desired ones
  * `# locale-gen`
  * Setup for `en_US` as language but `en_GB` specifics for dd/mm/yyyy, A4 paper and metric system
  * `# echo LANG=en_US.UTF-8 >> /etc/locale.conf`
  * `# echo LC_TIME=en_GB.UTF-8 >> /etc/locale.conf`
  * `# echo LC_PAPER=en_GB.UTF-8 >> /etc/locale.conf`
  * `# echo LC_MEASUREMENT=en_GB.UTF-8 >> /etc/locale.conf`
1. Set console font: `# echo KEYMAP=us-acentos /etc/vconsole.conf`
1. Set hostname: `# echo hostname > /etc/hostname`
1. Generate initial ramdisk environment:
  * `# vim /etc/mkinitcpio.conf`
  * Add `encrypt` and `btrfs` to `HOOKS` before filesystems, it should look like this:

        HOOKS=(base keyboard udev autodetect modconf block keymap encrypt btrfs filesystems resume)

  * `# mkinitcpio -p linux`
1. Set the root password: `# passwd`
1. Install GRUB: `# grub-install`
1. Configure GRUB:
  * Edit `# vim /etc/default/grub` and set `GRUB_CMDLINE_LINUX="cryptdevice=UUID=XXX-XXX-XXX:luks"`
    You can use `lsblk -f` to get the the uuid for /dev/nvmeXp2.
  * `# grub-mkconfig -o /boot/grub/grub.cfg`
1. Create a sudo user:
  * `# useradd -m -G wheel USERNAME`
  * `# passwd USERNAME`
  * `# visudo` and uncomment `%wheel ALL=(ALL) ALL`
1. Generate machine id: `# dbus-uuidgen --ensure`

## Setup custom desktop environment and reboot

As i'm using Gnome I install it before rebooting, so after the reboot we'll be
welcomed by a GUI to be able to connect to the internet and keep customizing
your system.

    # pacman -S gnome networkmanager
    # systemctl enable NetworkManager

After that exit the new system and reboot.

Exit new system and reboot:

    # exit
    # umount -R /mnt
    # reboot

You should be greeted by gdm and start enjoying your install.

