## Artix Linux Installation

Laptop: ASUS Vivobook S16 / Intel CORE ULTRA 7 / Intel ARC / 32Gb / 1TB

Artix Live Image: artix-xfce-s6

Goals:
- encrypted `/`, isolated `/home`
- OpenRC instead of the suspecious systemd
- Custom kernel - `linux-zen`
- EFI-only bootloader (no MBR)
- lightdm, i3wm, i3status, dmenu
- i915 support
- WiFi connection
- High DPI display support
- Audio working, volume controls working
- Brightness controls working
- Hibernation support
- Hardware acceleration

#### Write image to USB flash drive (Mac):

```bash
diskutil list
# /dev/disk4 - is the target, we add the "r" prefix to the block device
# unmount before writing
diskutil unmountDisk /dev/disk4
# write ISO
sudo dd if=/path-to/Downloads/artix-xfce-s6-20250407-x86_64.iso of=/dev/rdisk4 bs=1m status=progress && sync
# eject
diskutil eject /dev/disk4
```

#### Live System GRUB fix for i915

(making XFCE minimally working)

```
i915.modeset=0
```

Booting....

> Artix default user/pass: artix/artix

Switch to root:

```bash
sudo su
```

#### NVMe Setup

> Block Device: /dev/nvme0n1

```bash
fdisk /dev/nvme0n1
```

Commands:

```
# commands (use l to get type No)
g                                        (add GPT label)
#   No        Size     Type
n   AUTO(1)   +300M    t        1        (EFI System)
n   AUTO(2)   +1G      DEFAULT  DEFAULT  (Linux Filesystem, boot)
n   AUTO(3)   DEFAULT  t        44       (Linux LVM, encrypted root)
w
```

Verify

```bash
fdisk -l
```

Format

```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.ext2 /dev/nvme0n1p2
```

Creating encrypted LVM:

```bash
cryptsetup -y --use-random luksFormat /dev/nvme0n1p3
# type YES
# enter passphase...

# open the container
cryptsetup luksOpen /dev/nvme0n1p3 cryptroot
# verify
lsblk

# partitioning the /dev/mapper/cryptroot

# physical volume
pvcreate /dev/mapper/cryptroot

# volume group
vgcreate crypt_disk /dev/mapper/cryptroot

# or open existing
# vgchange -ay crypt_disk

# logical volumes (SWAP is required for hibernation)
lvcreate -n root -L 100G crypt_disk
lvcreate -n swap -L 36G crypt_disk # ~ RAM + 4G
mkswap /dev/mapper/crypt_disk-swap
swapon /dev/mapper/crypt_disk-swap
lvcreate -n home -l 100%FREE crypt_disk
# verify
lsblk

# format logical volumes
mkfs.ext4 /dev/crypt_disk/root
mkfs.ext4 /dev/crypt_disk/home
# verify
fdisk -l

# mount volumes to the system we're installing at /mnt

mount /dev/crypt_disk/root /mnt
mkdir -p /mnt/boot
mount /dev/nvme0n1p2 /mnt/boot
mkdir -p /mnt/boot/EFI
mount /dev/nvme0n1p1 /mnt/boot/EFI
mkdir -p /mnt/home
mount /dev/crypt_disk/home /home
```

#### Artix Installation

```bash
basestrap -i /mnt base base-devel \
 linux-zen linux-zen-headers \ # linux-zen kernel
 cryptsetup lvm2 device-mapper dosfstools \ # luks/fs
 grub efibootmgr os-prober \ # bootloader
 openrc elogind-openrc \ # OpenRC
 linux-firmware intel-ucode sof-firmware \ # Hardware support
 nano curl wget git ufw htop mc # tools

# Ignore mkinitcpio warnings like
# "Possibly missing firmware for ast,... xhci_pci_renesas, wd719x qed aic94xx qla2xxx qla1280 bfa"
# - they belong to some server hardware which is defined in linux-firmware but missing in our laptop
```

Creating fstab

```
fstabgen -Up /mnt >> /mnt/etc/fstab
# verify
cat /mnt/etc/fstab
```

#### chroot'ing to /mnt

```bash
artix-chroot /mnt
```
##### Base Configuration

```bash
# configuring locale.gen
nano /etc/locale.gen
# enable en_US.UTF-8, your locale...
# and generate the locales
locale-gen

# configuring timezone, for eg.
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
# sync time
hwclock --systohc

# updatw locale.conf
nano /etc/locale.conf
# set your locale, for eg.
LANG=en_US.UTF-8

# configuring vconsole
nano /etc/vconsole.conf
KEYMAP=us
FONT=solar24x32 # ! better for our High DPI display until we setup xrandr

# set hostname
nano /etc/hostname
artix

# configure hosts
nano /etc/hosts
127.0.0.1 localhost 
::1 localhost
127.0.0.1 artix.localdomain artix
```

##### initramfs

```bash
# configuring mkinitcpio
nano /etc/mkinitcpio.conf
# early load for ext4,
# and i915 - trying to fix hibernation/wakeup
MODULES(i915 ext4)
# before "filesystems", add "encrypt lvm2 resume",
# keep the right order !
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 resume filesystems fsck)

# LUKS support in GRUB
# find and copy UUID of the encrypted partition
blkid /dev/nvme0n1p3
# then edit /etc/default/grub, edit the defaults, replace the TheUUID
nano /etc/default/grub
# "quiet" is optional
GRUB_CMDLINE_LINUX_DEFAULT="log-level=3 cryptdevice=UUID=TheUUID:cryproot:allow-discards root=/dev/mapper/crypt_disk-root resume=/dev/mapper/crypt_disk-swap i915.enable_psr=0"
# GRUB_PRELOAD_MODULES="lvm" # required if /boot is encrypted, not in our case,
# lvm is activated later in initramfs

# run mkinitcpio, for linux-zen
mkinitcpio -p linux-zen
```

##### Install GRUB

```bash
grub-install --target=x86_64-efi --bootloader-id=GRUB --recheck

# optional
# cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo

# make GRUB config
grub-mkconfig -o /boot/grub/grub.cfg
```

##### Configuring users

```bash
pacman -S sudo bash

# set root password
passwd

# adding another user
useradd -m -G wheel,audio,video,input,storage,power -s /bin/bash theusername 
passwd theusername
nano /etc/sudoers
# uncomment the %wheel group
```

##### Drivers

```bash
# Intel ARC
# (!) DO NOT INSTALL xf86-video-intel!
# Using modesetting driver (embedded into xorg-server) 
# Mesa и Vulkan 
pacman -S mesa vulkan-intel intel-media-driver 
# GPU monitoring, optional -- from Arch repos (later)
# pacman -S intel-gpu-tools libva-utils
```

```bash
# Power management
# ! needed for Intel Core Ultra 7
pacman -S thermald thermald-openrc
# TLP - for laptops
pacman -S tlp tlp-openrc
# adding to OpenRC
rc-update add thermald default
rc-update add tlp default
```

##### Network

```bash
# Network + OpenRC daemons
pacman -S networkmanager networkmanager-openrc wpa_supplicant
# GUI tools
pacman -S network-manager-applet nm-connection-editor
# delete dhcpcd conflicting with nm
rc-update del dhcpcd default 2>/dev/null || true
# autoload nm
rc-update add NetworkManager default
```

##### Intel ARC Hardware Acceleration

```bash
# env for Firefox
nano /etc/environment
# add
# Intel ARC hardware acceleration
LIBVA_DRIVER_NAME=iHD
MOZ_X11_EGL=1
```

##### tmpfs

```bash
# add tmpfs to fstab
nano /etc/fstab
# simple, not secure
# tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0
# secure + max size
tmpfs /tmp tmpfs rw,nosuid,nodev,relatime,size=4G,mode=1777 0 0
```

##### Add Arch repos

```bash
# add pacman Arch repos
pacman -Syu artix-archlinux-support
nano /etc/pacman.conf
----
# enable 32bit id necessary
# uncomment [lib32]
# add Arch repos
[extra]
Include = /etc/pacman.d/mirrorlist-arch 
#[community]
#Include = /etc/pacman.d/mirrorlist-arch 
[multilib]
Include = /etc/pacman.d/mirrorlist-arch
----
nano /etc/pacman.d/mirrorlist-arch
# uncomment some servers...
# update keys
pacman-key --populate archlinux
# sync
pacman -Sy
```

##### X, i3w, lightdm, dmenu, etc. + OpenRC daemons

```bash
# X
pacman -S xorg-server xorg-xinit xorg-xrandr
# additional tools
pacman -S xorg-xsetroot xorg-xbacklight

# i3m status, dmenu
pacman -S i3-wm i3status i3lock dmenu
# rofi - alt launcher, but dmenu is better IMHO
# composer
pacman -S picom
# wallpapers
pacman -S feh nitrogen
# terminal
pacman -S alacritty
# file manager
#pacman -S thunar gvfs # alt
pacman -S ranger # vim-like
# screenshots
pacman -S flameshot
# theme manager 
pacman -S lxappearance
# firefox - default browser
pacman -S firefox
# fonts
pacman -S ttf-dejavu ttf-liberation noto-fonts ttf-font-awesome

# lightdm & greeter
pacman -S lightdm lightdm-gtk-greeter
# lightdm/OpenRC
pacman -S lightdm-openrc

rc-update add lightdm default
```




