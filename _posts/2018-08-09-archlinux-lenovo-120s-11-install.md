---
layout: post
title:  "Installing Arch Linux on a Lenovo 120s 11.6 (11IAP-129)"
date:   2018-08-09 01:01:01 +0200
categories: archlinux lenovo laptop
---

This a very concise guide to get Arch Linux up and running from scratch on the Lenovo 120s 11.6" model. It is based on an Arch install guide [publicly available here](https://github.com/maximbaz/dotfiles/blob/master/INSTALL_ARCH.md).
This model in particular is the one with 32 GB eMMC internal storage.
The steps consider wiping totally the factory Windows and use the internal drive fully for the Linux install.

## UEFI

1. Disable Secureboot (F2 on boot to access the setup)

## Create Arch install USB

1. [Download](https://archlinux.org/download/) ISO and sig files
2. Verify the image: `$ pacman-key -v archlinux-<version>-x86_64.iso.sig`
3. Create a bootable usb with:

        # dd bs=4M if=archlinux-*-x86_64.iso of=/dev/sdX status=progress oflag=sync

## Boot from USB

1. Get online `# wifi-menu`
2. Enable NTP: `# timedatectl set-ntp true`

## Preinstall details and backup of the factory partitions

Initially the partition scheme of the internal eMMC drive is the following:

    # lsblk
    mmcblk0      179:0    0  29.1G  0 disk
    ├─mmcblk0p1  179:1    0   260M  0 part
    ├─mmcblk0p2  179:2    0    16M  0 part
    ├─mmcblk0p3  179:3    0  27.9G  0 part
    └─mmcblk0p4  179:4    0  1000M  0 part

More details about the factory partitions:

    # parted /dev/mmcblk0
    GNU Parted 3.2
    Using /dev/mmcblk0
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) p
    Model: MMC HBG4a2 (sd/mmc)
    Disk /dev/mmcblk0: 31.3GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags:

    Warning: failed to translate partition name
    Number  Start   End     Size    File system  Name                          Flags
     1      1049kB  274MB   273MB   fat32        EFI system partition          boot, esp
     2      274MB   290MB   16.8MB               Microsoft reserved partition  msftres
     3      290MB   30.2GB  29.9GB  ntfs         Basic data partition          msftdata
     4      30.2GB  31.3GB  1049MB  ntfs                                       hidden, diag

As Windows is using EFI to boot the partition table is gpt already. The first partition is the EFI one, the second the Windows 10 required Microsoft reserved, the third is the main partition with Windows 10 installed and the last a 1GB hidden one, probably with some recovery purpose.

I'll wipe 2, 3 and 4 and keep the factory EFI partition, we need one for the Linux install and leaving the Windows stuff there won't hurt, 273MB is big enough (the full used space after the Linux install will be just 29MB).
But before I'll make a backup of the full eMMC drive in its current state.

Connect and mount an external drive to `/mnt/extdrive` with enough free space, the image clone of the eMMC drive will be of the same size, so we need at least 32GB in this case.

Make sure that all the eMMC partitions are unmounted and save a dump of the partition table:

    # fdisk -l /dev/mmcblk0 > /mnt/extdrive/emmc_partitions.info

Now we make the image clone of the drive with `dd`:

    # dd conv=sync,noerror bs=1M if=/dev/mmcblk0  of=/mnt/extdrive/backup_120s.img status=progress

This image can be applied back at any time[link to arch wiki] to the eMMC drive to restore it completely to factory state.


## Partitioning for Linux using an encrypted LVM for `/`

1. Partition the disk:
  * `# gdisk /dev/mmcX`
  * keep 1 the EFI partition   (EF00)
  * delete 2, 3 and 4
  * create a partition with 250M for `/boot` with Linux filesystem (8300)
  * create another using the remaining space for Linux LVM  (8e00)
2. Only in case the drive changes are not captured with `# lsblk` reboot.
3. Create the filesystem for `/boot`
  * `# mkfs.ext4 /dev/mmcXp2`
4. Encrypt the LVM partition:
  * `# cryptsetup -v luksFormat /dev/mmcXp3`
  * `# cryptsetup luksOpen /dev/mmcXp3 luks`
5. Create a physical volume: `# pvcreate /dev/mapper/luks`
6. Create a volume group: `# vgcreate arch /dev/mapper/luks`
7. Create logical volumes, I'll be using all for `/`:
  * `# lvcreate -l 100%FREE arch -n root`
8. Create file systems on the logical volume:
  * `# mkfs.ext4 /dev/mapper/arch-root`
9. Mount logical volumes:
  * `# mount /dev/mapper/arch-root /mnt`
  * `# mkdir /mnt/boot`
  * `# mount /dev/mmcXp2 /mnt/boot`
  * `# mkdir /mnt/boot/efi`
  * `# mount /dev/mmcXp1 /mnt/boot/efi`
10. Help new system discover LVM configuration made on host:
  * `# mkdir /mnt/hostrun`
  * `# mount --bind /run /mnt/hostrun`

## Install Arch

1. Install basic packages:

        # pacstrap /mnt base base-devel
                        grub efibootmgr os-prober   # boot stuff
                        intel-ucode                 # for intel cpus
                        vim git openssh ansible     # tools to deploy playbooks
                        dialog wpa_supplicant       # to use wifi-config on reboot

1. Generate fstab entries: `# genfstab -U /mnt >> /mnt/etc/fstab`
1. Open `# vim /mnt/etc/fstab` and make the following adjustments:
  * Change `relatime` to `noatime` for all non-boot partitions to improve durability of SSD.
1. Enter the new system: `# arch-chroot /mnt`
1. Bind LVM configuration from host:
  * `# mkdir /run/lvm`
  * `# mount --bind /hostrun/lvm /run/lvm`
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
  * Add `encrypt` and `lvm2` to `HOOKS` before filesystems
  * `# mkinitcpio -p linux`
1. Set the root password: `# passwd`
1. Install GRUB: `# grub-install`
1. Configure GRUB:
  * Edit `# vim /etc/default/grub` and set `GRUB_CMDLINE_LINUX="cryptdevice=/dev/sdx3:luks"`
  * `# grub-mkconfig -o /boot/grub/grub.cfg`
1. Create a sudo user:
  * `# useradd -m -G wheel USERNAME`
  * `# passwd USERNAME`
  * `# visudo` and uncomment `%wheel ALL=(ALL) ALL`
1. Generate machine id: `# dbus-uuidgen --ensure`
1. Exit new system and reboot:
  * `# exit`
  * `# umount -R /mnt`
  * `# reboot`

## Setup custom environment

I'm using i3 as the main graphical environment with the i3-gnome project to
handle all the system services (automount usb drives, sound settings, bluetooth
pairing, ...).

The setup is based on two very simple ansible playbooks, the first will install
several packages including a basic Gnome desktop environment and the i3 window
manager.  At the end it will download the required sources from the AUR to
build the `pikaur` helper. The build and install should be done manually as
this can require specific manual intervention.

The second playbook is just the dotfiles setup for the local user to customize
the bash environment and vim.

1. Login with the new user
1. Get the first playbook:
  * `$ curl -Ls https://raw.githubusercontent.com/arielzn/dotfiles/master/arch_install/ansible_init_setup.yml`
1. Run it with become option: `$ ansible-playbook -b -K ansible_init_setup.yml`
1. Compile downloaded sources for `pikaur` helper:
  * `$ cd /tmp`
  * `$ tar xf cower.tar.gz`
  * `$ tar xf pikaur.tar.gz`
  * `$ cd cower/ ; makepkg`
  * `$ sudo pacman -U cower-*.pkg.tar.xz`
  * `$ cd ../pikaur/ ; makepkg`
  * `$ sudo pacman -U pikaur-*.pkg.tar.xz`
  * If there are issues with signatures run `gpg --recv-keys --keyserver hkp://pgp.mit.edu XXXXXXXXXXX` for the missing key and retry
1. Install some AUR packages:
  * `$ pikaur -S powerline-fonts-git i3-gnome py3status python-pydbus byobu google-chrome`
1. Reboot: `$ sudo reboot`
1. Setup local user environment:
  * `$ curl -Ls https://raw.githubusercontent.com/arielzn/dotfiles/master/arch_install/ansible_user_env.yml`
  * `$ ansible-playbook ansible_user_env.yml`

The install finishes with a bit less than 6GB used in `/` not bad for a complete DE with all the basics apps in place (Firefox and Chrome browsers, Office Apps, Photo editing and Media reproduction).

