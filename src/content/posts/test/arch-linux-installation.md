---
title: Arch Linux Installation
published: 2025-09-21
tags:
  - Linux
  - Arch
pin: 99
---

![Image Description](https://archlinux.org/static/logos/archlinux-logo-dark-90dpi.png)

**YouTube Tutorial** - https://www.youtube.com/watch?v=5DHz23VQJxk

**Arch Wiki Official Install** - https://wiki.archlinux.org/title/Installation_guide
## 1. ISO Console settings

 To set console font size
 All console fonts location -`/usr/share/kbd/consolefonts/`

```shell
setfont ter-132b
```

## 2.  Internet Setup

Use ping command to check if internet connection is available or not

```shell
ping google.com
```

If getting packets Hurry!
If not getting follow below guide:

------------------------------------------------------------------------

Use [iwctl](https://wiki.archlinux.org/title/Iwd#iwctl) CLI utility

To get an interactive prompt do:

```shell
iwctl
```

Now a interactive prompt will open prefixed with `[iwd]#`

To list all available commands:

```shell
help
```

First, if you do not know your wireless device name, list all Wi-Fi devices:

```shell
device list
```

If the device or its corresponding adapter is turned off, turn it on:

```shell
device {name} set-property Powered on
```

```shell
adapter {adapter} set-property Powered on
```

Then, to initiate a scan for networks (note that this command will not output anything):

```shell
station {name} scan
```

You can then list all available networks:

```shell
station {name} get-networks
```

Finally, to connect to a network:

```shell
station {name} connect {SID}
```

---

```
nmcli device wifi list

nmcli device wifi connect {network_name} password {password}
```
## 3. Update the system clock

Use [timedatectl(1)](https://man.archlinux.org/man/timedatectl.1) to ensure the system clock is synchronized:

```shell
timedatectl
```

## 4. Partition the disks

When recognized by the live system, disks are assigned to a [block device](https://wiki.archlinux.org/title/Block_device "Block device") such as `/dev/sda`, `/dev/nvme0n1` or `/dev/mmcblk0`. To identify these devices, use [lsblk](https://wiki.archlinux.org/title/Lsblk "Lsblk") or _fdisk_.

```shell
fdisk -l
```

Or

```shell
lsblk
```

Create partitions using [cfdisk](https://man.archlinux.org/man/cfdisk.8) or [fdisk](https://wiki.archlinux.org/title/Fdisk)

Create following type of layout according to your firmware

> [!tip]
> Your can create additional partitions like `/home` etc for more easy data retrieval in case of system failure

> [!tip]
> You can create swap on ZRAM which is very fast compared to traditional swap approaches.
> If you really want a swap but not on the ZRAM use swap files instead of a partition. These files can be created and removed at system runtime which is not possible in case of partitions as those need to be unmounted first

![[Pasted image 20241129080319.png]]

## 5. Format the partitions

Once the partitions have been created, each newly created partition must be formatted with an appropriate [file system](https://wiki.archlinux.org/title/File_system "File system"). See [File systems#Create a file system](https://wiki.archlinux.org/title/File_systems#Create_a_file_system "File systems") for details.

Use [mkfs(8)](https://man.archlinux.org/man/mkfs.8.en) CLI utility :

### Linux File Systems

> [!info]
> Format other partitions created like `/home` with ext4 format or other format which are linux supported otherwise system will not read other partitions

```shell
mkfs.ext4 /dev/{root_partition}
```

### Swap

If you created a partition for [swap](https://wiki.archlinux.org/title/Swap "Swap"), initialize it with [mkswap(8)](https://man.archlinux.org/man/mkswap.8):

```shell
mkswap /dev/{swap_partition}
```

### EFI System Partition

>[!caution]
> Only format the EFI system partition if you created it during the partitioning step. If there already was an EFI system partition on disk beforehand, reformatting it can destroy the boot loaders of other installed operating system

If you created an EFI system partition, [format it](https://wiki.archlinux.org/title/EFI_system_partition#Format_the_partition "EFI system partition") to FAT32 using [mkfs.fat(8)](https://man.archlinux.org/man/mkfs.fat.8).

```shell
mkfs.fat -F 32 /dev/{efi_system_partition}
```

## 6. Mount File Systems

> [!caution]
> Mount partitions in hierarchical order (First Parent Folder than Child Folder). Otherwise [genfstab(8)](https://man.archlinux.org/man/genfstab.8) wont be able properly generate the file and will probably miss some mount points.
>
> Ex: If wants to mount system in `/mnt` follow:
> 1. First mount root file system to `/mnt`
> 2. Now mount other child file systems under `/mnt` in hierarchical order
>
>   Same applies when wants to mount file systems in nested directories (First Parent Then Child)

[Mount](https://wiki.archlinux.org/title/Mount "Mount") the root volume to `/mnt`. For example, if the root volume is `/dev/{root_partition}`:

```shell
mount /dev/{root_partition} /mnt
```

Create any remaining mount points under `/mnt` (such as `/mnt/boot` for `/boot`) and mount the volumes in their corresponding hierarchical order.

> [!tip]
> Run [mount(8)](https://man.archlinux.org/man/mount.8) with the `--mkdir` option to create the specified mount point. Alternatively, create it using [mkdir(1)](https://man.archlinux.org/man/mkdir.1) beforehand.

For UEFI systems, mount the EFI system partition:

```shell
mount --mkdir /dev/{efi_system_partition} /mnt/boot
```

If you created a [swap](https://wiki.archlinux.org/title/Swap "Swap") volume, enable it with [swapon(8)](https://man.archlinux.org/man/swapon.8):

```shell
swapon /dev/{swap_partition}
```

[genfstab(8)](https://man.archlinux.org/man/genfstab.8) will later detect mounted file systems and swap space.

## 7. Select the Mirrors

Packages to be installed must be downloaded from [mirror servers](https://wiki.archlinux.org/title/Mirrors "Mirrors"), which are defined in `/etc/pacman.d/mirrorlist`. On the live system, after connecting to the internet, [reflector](https://wiki.archlinux.org/title/Reflector "Reflector") updates the mirror list by choosing 20 most recently synchronized HTTPS mirrors and sorting them by download rate.

The higher a mirror is placed in the list, the more priority it is given when downloading a package. You may want to inspect the file to see if it is satisfactory. If it is not, [edit](https://wiki.archlinux.org/title/Textedit "Textedit") the file accordingly, and move the geographically closest mirrors to the top of the list, although other criteria should be taken into account.

This file will later be copied to the new system by _pacstrap_, so it is worth getting right.

**Ex:**

```shell
reflector --country India --latest 5 --sort rate --save /etc/pacman.d/mirrorlist
```

## 8.  Install essential packages

> [!info]
> No software or configuration (except for `/etc/pacman.d/mirrorlist`) gets carried over from the live environment to the installed system.

Use the [pacstrap(8)](https://man.archlinux.org/man/pacstrap.8) script to install the [base](https://archlinux.org/packages/?name=base) package, Linux [kernel](https://wiki.archlinux.org/title/Kernel "Kernel") and firmware for common hardware:

```shell
pacstrap -K /mnt base linux linux-firmware
```

The [base](https://archlinux.org/packages/?name=base) package does not include all tools from the live installation, so installing more packages may be necessary for a fully functional base system. To install other packages or package groups, append the names to the _pacstrap_ command above (space separated) or use [pacman](https://wiki.archlinux.org/title/Pacman "Pacman") to [install](https://wiki.archlinux.org/title/Install "Install") them while [chrooted into the new system](https://wiki.archlinux.org/title/Installation_guide#Chroot)

### Recommended  packages
- [Network Manager](https://wiki.archlinux.org/title/NetworkManager)

**Ex:**

```shell
pacstrap -K /mnt base linux linux-firmware networkmanager
```

## 9. Configure the System

### 1. **Fstab**

Generate an [fstab](https://wiki.archlinux.org/title/Fstab "Fstab") file (use `-U` or `-L` to define by [UUID](https://wiki.archlinux.org/title/UUID "UUID") or labels, respectively):

```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

Check the resulting `/mnt/etc/fstab` file, and [edit](https://wiki.archlinux.org/title/Textedit "Textedit") it in case of errors.

### 2. **Chroot**

[Change root](https://wiki.archlinux.org/title/Change_root "Change root") into the new system:

```shell
arch-chroot /mnt
```

### 3. **Time**

Set the [time zone](https://wiki.archlinux.org/title/Time_zone "Time zone"):

```shell
ln -sf /usr/share/zoneinfo/{Region}/{City} /etc/localtime
```

Run [hwclock(8)](https://man.archlinux.org/man/hwclock.8) to generate `/etc/adjtime`:

```shell
hwclock --systohc
```

### 4. **Localization**

[Edit](https://wiki.archlinux.org/title/Textedit "Textedit") `/etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8` and other needed UTF-8 [locales](https://wiki.archlinux.org/title/Locale "Locale"). Generate the locales by running:

```shell
locale-gen
```

[Create](https://wiki.archlinux.org/title/Create "Create") the [locale.conf(5)](https://man.archlinux.org/man/locale.conf.5) file, and [set the LANG variable](https://wiki.archlinux.org/title/Locale#Setting_the_system_locale "Locale") accordingly:

```shell
/etc/locale.conf
```

```shell
LANG=_en_US.UTF-8_
```

### 5. **Network Configuration**

[Create](https://wiki.archlinux.org/title/Create "Create") the [hostname](https://wiki.archlinux.org/title/Hostname "Hostname") file:

```shell
/etc/hostname
```

```shell
{yourhostname}
```

### **6. Root Password**

Set the root [password](https://wiki.archlinux.org/title/Password "Password"):

```shell
passwd
```

### **7. Add User Account**
```
useradd -m -G wheel amit
passwd amit
```

### **8. Services**
```
systemctl enable NetworkManager
```

### **9. GRUB (Bootloader)**

Install [grub](https://archlinux.org/packages/?name=grub) and [efibootmgr](https://archlinux.org/packages/?name=efibootmgr): _GRUB_ is the boot loader while _efibootmgr_ is used by the GRUB installation script to write boot entries to NVRAM.

Mount the EFI Partition

```shell
mount --mkdir /dev/{efi_system_partition} /mnt/boot
```

> [!Caution] Note:
> - Make sure to install the packages and run the `grub-install` command from the system in which GRUB will be installed as the boot loader. That means if you are booting from the live installation environment, you need to be inside the chroot when running `grub-install`. If for some reason it is necessary to run `grub-install` from outside of the installed system, append the `--boot-directory=` option with the path to the mounted `/boot` directory, e.g `--boot-directory=/mnt/boot`.
> - Some motherboards cannot handle `bootloader-id` with spaces in it.

```shell
grub-install --target=x86_64-efi --bootloader-id=Arch
```

> [!NOTE] Note
> If for some reason grub-install not able to find EFI directory & throwing below error follow this guide:
>
> **Error** - `grub-install: error: cannot find EFI directory`
> **Solution** - Append `--efi-directory=/boot` flag to explicitly tell grub about EFI directory

Now generate grub config in boot directory so grub can find linux kernel entries on boot time

```
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Dual Boot

If you want grub to search for other installed OS automatically & list them in grub menu install [os-prober](https://archlinux.org/packages/?name=os-prober) pacakage and mount the partitions from which the other systems boot.
Then re-run  _grub-mkconfig_ command.

If you get the following output:
`Warning: os-prober will not be executed to detect other bootable partitions`

Then edit `/etc/default/grub` and add/uncomment:

```
GRUB_DISABLE_OS_PROBER=false
```

## Reboot

Exit the chroot environment by typing `exit` or pressing `Ctrl+d`.

Optionally manually unmount all the partitions with `umount -R /mnt`: this allows noticing any "busy" partitions, and finding the cause with [fuser(1)](https://man.archlinux.org/man/fuser.1).

Finally, restart the machine by typing `reboot`: any partitions still mounted will be automatically unmounted by _systemd_. Remember to remove the installation medium and then login into the new system with the root account.

## Post Installation

### Network Manager

```
systemctl start NetworkManager
systemctl enable NetworkManager
```

### Add new User

```
useradd -m -G wheel amit
```

### Setup Sudo

```
pacman -S sudo
EDITOR=nano visudo
```

When sudoers file opens uncomment `wheel` group code

### Setup Flatpak
```
pacman -S flatpak
```

### Setup KDE Plasma
```
pacman -S plasma-meta konsole dolphin firefox
systemctl enable sddm
systemctl start sddm
```

### Multi Lib Repository

[Multilib Doc](https://wiki.archlinux.org/title/Official_repositories#multilib)

_multilib_ contains 32-bit software and libraries that can be used to run and build 32-bit applications on 64-bit installs (e.g. [wine](https://archlinux.org/packages/?name=wine), [steam](https://archlinux.org/packages/?name=steam), etc).

#### Enable
To enable _multilib_ repository, uncomment the `[multilib]` section in `/etc/pacman.conf`:
```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

> [!Info] Tip
> **Tip:** Run `pacman -Sl multilib` to list all packages in the _multilib_ repository. 32-bit library package names begin with `lib32-`.

#### Disable

Execute the following command to remove all packages that were installed from _multilib_:
```
pacman -R $(comm -12 <(pacman -Qq | sort) <(pacman -Slq multilib | sort))
```

Comment out the `[multilib]` section in `/etc/pacman.conf`:
```
#[multilib]
#Include = /etc/pacman.d/mirrorlist
```

## Graphics Drivers

### In Kernel
- amdGPU kernel - Already in Kernel

### 1. OpenGL

> [!Info] Tip
>  For AMD (and ATI) it is recommended to use the open-source drivers unless you have a strong reason to use the proprietary ones.

#### 1.1 AMD
- amdgpu-pro-oglp - (Pro) Closed Source (Proprietary)

#### 1.2 Open Source
- mesa - 64 Bit Open Source
- lib32-mesa - 32 Bit Open Source
- ~~libva-mesa-driver~~ - already in mesa
- ~~libva-utils~~ - already in mesa
### 2. Vulkan

#### 2.1 AMD
- amdvlk - 64 Bit Open Source
- lib32-amdvlk - 32 Bit Open Source
- vulkan-amdgpu-pro - 64 Bit Closed Source
- lib32-vulkan-amdgpu-pro - 32 Bit Closed Source

#### 2.2 Open Source
- vulkan-radeon (RADV) - 64 Bit (included in mesa package)
- lib32-vulkan-radeon (RADV) - 32 Bit (included in lib32-mesa package)

Chaotic AUR

## Temp

## TTY

`ctrl + alt + f6`

```
loginctl list-session
```

```
loginctl kill-session {session_id}
```
