*This guide makes insane, non-explicit assumptions (aka be me) so do not use it if you are not me!*

We use the [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64) as a reference

## Table of contents

- [Preparing the disks](#preparing-the-disks)
- [Installing stage3](#installing-stage3)
- [Configure Portage](#configure-portage)
- [Preparing the chroot environement](#preparing-the-chroot-environement)
- [Installing the base system](#installing-the-base-system)
- [Post-installation tasks](#post-installation-tasks)
- [systemd bits](#systemd-bits)
- [Configuring the kernel](#configuring-the-kernel)
- [Install the full desktop environment](#install-the-full-desktop-environment)
- [Configuring the system](#configuring-the-system)
- [Configuring the bootloader](#configuring-the-bootloader)
- [Finalizing setup](#finalizing-setup)

## Preparing the disks

Layout proposal (swap size might be overkill)

Block device    | Description                 | Filesystem    | Size
----------------|-----------------------------|---------------|-------------
/dev/nvme0n1p1  | EFI system partition (ESP)  | vFAT (FAT32)  | 512 MiB
/dev/nvme0n1p2  | Swap partition              | N/A           | 20 GiB
/dev/nvme0n1p3  | Root filesystem partition   | Ext4          | Rest of the disk

First, from the livecd, take superadmin powers with `sudo su -`

Make new partitions with fdisk
```bash
fdisk -l /dev/nvme0n1
fdisk /dev/nvme0n1
```

It should look like this
```
Command (m for help): p
Disk /dev/nvme0n1: 953.87 GiB, 1024209543168 bytes, 2000409264 sectors
Disk model: SKHynix_HFS001TDE9X084N                 
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 6B9A7892-118B-904F-A57B-7D050AAD25DF

Device            Start        End    Sectors   Size Type
/dev/nvme0n1p1     2048    1050623    1048576   512M EFI System
/dev/nvme0n1p2  1050624   42993663   41943040    20G Linux swap
/dev/nvme0n1p3 42993664 2000409230 1957415567 933.4G Linux root (x86-64)

Filesystem/RAID signature on partition 1 will be wiped.
```

Make filesystems
```bash
mkfs.vfat -F 32 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p3
```

Swap
```bash
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2
```

Check everything is okay with `lsblk -f`

## Installing stage3

Mount the root partition
```bash
mkdir -p /mnt/gentoo
mount /dev/nvme0n1p3 /mnt/gentoo
```

Download the [latest stage tarball](https://www.gentoo.org/downloads/#other-arches), we're going for `desktop | systemd | mergedusr`
```bash
cd /mnt/gentoo
wget <PASTED_STAGE_URL>
```

And extract it
```bash
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

## Configure Portage

Select mirrors with this nice tool
```bash
mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
```

Fire up good ol' nano
```bash
nano -w /mnt/gentoo/etc/portage/make.conf
```

Add `-march=native` to `COMMON_FLAGS` and call it a day

Additionally, append these lines to `make.conf`
```bash
# Self-explanatory
EMERGE_DEFAULT_OPTS="--ask --verbose --quiet-build"

# USE variable
USE="dist-kernel"

ACCEPT_LICENSE="@BINARY-REDISTRIBUTABLE"
VIDEO_CARDS="nvidia"
```

## Preparing the chroot environement

Configure the base repo
```bash
mkdir -p /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

Copy DNS info, probably shouldn't do this but it's fine
```bash
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

Mounting the necessary filesystems, [read this](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Base#Mounting_the_necessary_filesystems) for more information
```bash
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
```

Finally, chroot in the new environment (yay!)
```bash
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

## Installing the base system

Get a snapshot of the Gentoo ebuild repo
```bash
emerge-webrsync
```

Do the basic maintenance tasks
```bash
eselect news list
eselect news read
```

Set desired profile (plasma/systemd/merged-usr)
```bash
eselect profile list
eselect profile set 11
```

Emerge the world set
```bash
emerge --ask --verbose --update --deep --newuse @world

# or simply
emerge -uDN @world
```

## Post-installation tasks

Install these utilities  
```bash
emerge cpuid2cpuflags bash-completion
```

Add `CPU_FLAGS_X86`  
```bash
echo "*/* $(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags
```

At this point it might be a good idea to remerge
```bash
emerge -uDN @world
```

## systemd bits

Set timezone
```bash
ln -sf ../usr/share/zoneinfo/Europe/Paris /etc/localtime
```

Configure locales
```bash
nano -w /etc/locale.gen

en_US.UTF-8 UTF-8
fr_FR.UTF-8 UTF-8
```

Followed by  
```bash
locale-gen
eselect locale list
eselect locale set 4
```

Finish with
```bash
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
```
As usual, the [Gentoo documentation](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Base#Locale_selection) is top notch if you need more detailled explanations

## Configuring the kernel

Install necessary packages for distribution kernels
```bash
emerge linux-firmware installkernel-gentoo
```

Build from source
```bash
emerge sys-kernel/gentoo-kernel
```

Or get the binary version
```bash
emerge sys-kernel/gentoo-kernel-bin
```

## Install the full desktop environment

Pull entire KDE Plasma
```bash
emerge kde-plasma/plasma-meta
```
Might take a while

## Configuring the system

Edit fstab
```bash
nano -w /etc/fstab

# <device>                                      <mnt>   <type>  <options>       <dump fsck>
# esp
PARTUUID="a5daf4a2-0f01-c946-82d1-9d7eb47f1334" none    vfat    defaults        0 2
# swap
PARTUUID="cf09424d-8807-d34c-ac3b-06c181621033" none    swap    defaults        0 0
# rootfs
PARTUUID="f700f734-fa98-494d-bafe-6b70aa36674d" /       ext4    defaults        0 1
```

More info [here](https://wiki.archlinux.org/title/fstab)

Set root password
```bash
passwd
```

If you get `Weak password: too short.`, change `/etc/security/passwdqc.conf` to
```bash
enforce=none
```

Init and boot configuration (systemd)
```bash
systemd-firstboot --prompt --setup-machine-id
systemctl preset-all --preset-mode=enable-only
```

A few other things
```bash
# for faster file indexing
emerge mlocate

# why not?
emerge dosfstools

# systemd services
systemctl enable sshd
systemctl enable systemd-timesyncd.service
```

## Configuring the bootloader

Mount the ESP  
```bash
mkdir -p /boot/efi
mount /dev/nvme0n1p1 /boot/efi
```

GRUB
```bash
emerge grub
grub-install --target=x86_64-efi --efi-directory=/boot/efi
```

## Finalizing setup

Sudo
```bash
emerge sudo
visudo
```

Add user
```bash
useradd -m -G users,wheel,audio,video -s /bin/bash anon
passwd anon
```

Necessary utilities after reboot
```bash
emerge konsole kwrite firefox-bin
```

Remove the tarballs
```bash
cd /
rm stage3-amd64-desktop-systemd-*
```

Exit the chroot
```bash
exit
cd
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
```

Brace yourself and
```bash
reboot
```
