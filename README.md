# Arch Linux Install Guide

_Personal AMD development machines — last updated June 2026_

> **Based on:**
>
> - Learn Linux TV (May 28, 2026) — _How to Install Arch Linux in 2026: The Complete Step-by-Step Guide_
> - Josean Martinez (May 5, 2026) — _The Only Arch Linux Installation Guide You'll Ever Need_
> - DistroTube (Feb 10, 2026) — _An Arch Linux Installation Guide (2026)_
> - The Rad Lectures (May 10, 2025) — _2025 Arch Linux Install with COSMIC | Full Step-by-Step Guide [BTRFS, Encrypt, Zram, Timeshift]_
> - Bread on Penguins (Sep 25, 2024) — _Beginner Friendly Arch Linux Installation Guide and Walkthrough_
> - DenshiVideo (Aug 18, 2024) — _Arch Linux: An Encrypted Guide_

---

## Table of Contents

1. [Pre-Installation](#1-pre-installation)
2. [Optional: SSH Into Your Target Machine](#2-optional-ssh-into-your-target-machine)
3. [Connect to Wi-Fi](#3-connect-to-wi-fi)
4. [Partition the Disk](#4-partition-the-disk)
5. [Encrypt and Format](#5-encrypt-and-format)
6. [Mount Filesystems](#6-mount-filesystems)
7. [Install the Base System](#7-install-the-base-system)
8. [Configure the System (chroot)](#8-configure-the-system-chroot)
9. [Install Essential Packages](#9-install-essential-packages)
10. [Configure mkinitcpio](#10-configure-mkinitcpio)
11. [Install and Configure GRUB](#11-install-and-configure-grub)
12. [Enable Services](#12-enable-services)
13. [First Boot](#13-first-boot)
14. [Post-Install: AUR Helper & Snapshots](#14-post-install-aur-helper--snapshots)
15. [Zram Setup](#15-zram-setup)
16. [Display Manager](#16-display-manager)
17. [Desktop Environment (COSMIC)](#17-desktop-environment-cosmic)
18. [AMD GPU Drivers](#18-amd-gpu-drivers)

---

## 1. Pre-Installation

Disable **Secure Boot** in BIOS/UEFI, then boot into the Arch ISO from USB.

```bash
setfont ter-132b              # increase terminal font size
localectl list-keymaps        # list available keymaps
loadkeys us                   # load your keymap
timedatectl list-timezones    # list available timezones
timedatectl set-timezone America/Phoenix
timedatectl set-ntp true
```

Verify you're in UEFI mode — should output `64`:

```bash
cat /sys/firmware/efi/fw_platform_size
```

---

## 2. Optional: SSH Into Your Target Machine

Connecting over **Ethernet** is strongly recommended — it's faster and more reliable than Wi-Fi.

Set a password for the ISO root user (required for SSH):

```bash
passwd
```

Check that `sshd` is running:

```bash
systemctl status sshd
```

If it isn't, start it:

```bash
systemctl start sshd
```

Find your IP address:

```bash
ip addr show
```

On a ThinkPad with Wi-Fi, look under `wlan0` for the first `inet` address (e.g. `192.168.x.xx`). If you're on Ethernet, check `eth0` or `enp*`.

On your other machine, connect via SSH:

```bash
ssh root@<your-ip>
```

> If your other machine doesn't have OpenSSH installed:

```bash
sudo pacman -S openssh
sudo systemctl enable --now sshd
```

If you're on Ethernet, skip the next section.

---

## 3. Connect to Wi-Fi

```bash
iwctl
```

Inside the `iwd` prompt (replace `wlan0` and `SSID` with yours):

```
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect SSID
```

Exit `iwctl`, then verify connectivity:

```bash
ping -c 3 archlinux.org
```

---

## 4. Partition the Disk

> **Caution:** This will erase everything on the target disk. Confirm you have the right device with `lsblk`.

```bash
gdisk /dev/nvme0n1
```

If prompted to remove an existing signature, press `Y`.

Press `o` to wipe all existing partitions, then create two new ones:

**EFI partition (2 GB):**

```
n → 1 → (default) → +2G → ef00
```

**Linux partition (rest of disk — LUKS container):**

```
n → 2 → (default) → (default) → (default)
```

Write changes and exit:

```
w
```

Verify the layout:

```bash
lsblk
```

---

## 5. Encrypt and Format

Format the Linux partition with LUKS:

```bash
cryptsetup luksFormat /dev/nvme0n1p2
```

Type `YES` (uppercase) and set a strong passphrase.

> **Warning:** Do not lose this passphrase. Without it, your data is permanently inaccessible.

Open the encrypted container:

```bash
cryptsetup luksOpen /dev/nvme0n1p2 main
```

Format the container as Btrfs and create subvolumes:

```bash
mkfs.btrfs /dev/mapper/main
mount /dev/mapper/main /mnt
cd /mnt

btrfs subvolume create @
btrfs subvolume create @home

ls          # verify both subvolumes were created
cd -

umount /mnt
```

---

## 6. Mount Filesystems

Mount the subvolumes with recommended options:

```bash
mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@ /dev/mapper/main /mnt

mkdir /mnt/home
mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@home /dev/mapper/main /mnt/home
```

Format the EFI partition and mount it:

```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

---

## 7. Install the Base System

```bash
pacstrap -K /mnt base linux linux-headers linux-lts linux-lts-headers linux-firmware sudo vi rsync
```

Generate the fstab:

```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab    # verify it looks correct
```

---

## 8. Configure the System (chroot)

Enter the new system:

```bash
arch-chroot /mnt
```

### Timezone & Clock

```bash
ln -sf /usr/share/zoneinfo/America/Phoenix /etc/localtime
hwclock --systohc
```

### Install Neovim

```bash
pacman -Syu neovim
```

### Enable Parallel Downloads

```bash
nvim /etc/pacman.conf
```

Uncomment `ParallelDownloads = 5` and raise the value to `8` (or match your CPU thread count).

### Locale

```bash
nvim /etc/locale.gen
```

Uncomment `en_US.UTF-8 UTF-8`, then generate and set it:

```bash
locale-gen
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
echo "KEYMAP=us" >> /etc/vconsole.conf
```

### Hostname

```bash
echo "mycomputer" >> /etc/hostname
```

### Root Password

```bash
passwd
```

### Create a User

```bash
useradd -m -g users -G wheel,audio,video,optical,storage,input yourusername
passwd yourusername
```

### Configure sudo

```bash
EDITOR=nvim visudo
```

Uncomment the following line to grant `wheel` group members sudo access:

```
%wheel ALL=(ALL:ALL) ALL
```

### Update Mirrors

```bash
pacman -Sy reflector
reflector --country US --latest 5 --sort rate --save /etc/pacman.d/mirrorlist
```

---

## 9. Install Essential Packages

```bash
# Core system
pacman -S base-devel git

# Filesystem + boot
pacman -S btrfs-progs grub efibootmgr dosfstools mtools

# Networking
pacman -S networkmanager network-manager-applet openssh

# Security
pacman -S iptables-nft ipset firewalld

# Hardware (AMD CPU microcode)
pacman -S acpid amd-ucode

# Snapshots
pacman -S grub-btrfs timeshift inotify-tools

# Memory compression
pacman -S zram-generator

# Documentation
pacman -S man-db man-pages man texinfo

# Bluetooth
pacman -S bluez bluez-utils

# Audio
pacman -S pipewire pipewire-jack pipewire-alsa pipewire-pulse wireplumber alsa-utils sof-firmware

# Applications
pacman -S kitty firefox ttf-firacode-nerd ttf-jetbrains-mono-nerd ly
```

---

## 10. Configure mkinitcpio

Edit the initramfs config to add Btrfs and LUKS support:

```bash
nvim /etc/mkinitcpio.conf
```

Update `MODULES` and `HOOKS` to match the following:

```ini
MODULES=(btrfs atkbd)   # atkbd allows passphrase entry over SSH
HOOKS=(base udev autodetect modconf kms keyboard keymap block sd-encrypt filesystems)
```

Rebuild the initramfs:

```bash
mkinitcpio -P
```

---

## 11. Install and Configure GRUB

Install GRUB to the EFI partition:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=WhateverYouWant /dev/nvme0n1
```

Append the UUID of your encrypted partition to the GRUB config file:

```bash
blkid -o value -s UUID /dev/nvme0n1p2 >> /etc/default/grub
```

Open the file and move the UUID into `GRUB_CMDLINE_LINUX_DEFAULT`:

```bash
nvim /etc/default/grub
```

Set it like this (replace `<uuid>` with the actual UUID you appended):

```ini
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=<uuid>:main root=/dev/mapper/main"
```

Generate the final GRUB config:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 12. Enable Services

```bash
systemctl enable NetworkManager    # network management
systemctl enable bluetooth         # Bluetooth
systemctl enable sshd              # SSH server
systemctl enable firewalld         # firewall
systemctl enable reflector.timer   # automatic mirror ranking
systemctl enable acpid             # ACPI events (lid close, power button, etc.)
systemctl enable fstrim.timer      # weekly SSD TRIM
systemctl enable ly@tty2.service   # display manager
```

Exit chroot and reboot:

```bash
exit
reboot
```

---

## 13. First Boot

At startup you'll see the GRUB menu. Select **Arch Linux**, enter your encryption passphrase, and `ly` will present the login screen. Log in with the user you created.

Connect to Wi-Fi if needed:

```bash
nmcli device wifi list
nmcli device wifi connect "SSID"    # wrap SSID in quotes if it contains spaces
ping -c 5 archlinux.org             # verify connectivity
```

---

## 14. Post-Install: AUR Helper & Snapshots

### Install paru (AUR helper)

```bash
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

### Configure Timeshift

```bash
paru -S timeshift timeshift-autosnap
sudo timeshift --list-devices
sudo timeshift --create --comments "Initial snapshot" --tags D # Example comment: "[12 May 2025] Start of Time"
```

### Configure grub-btrfsd to Use Timeshift Snapshots

```bash
sudo su
cd ~
echo "export EDITOR=nvim" > .bashrc
source .bashrc
sudo systemctl edit --full grub-btrfsd
```

In the editor, replace:

```
ExecStart=/usr/bin/grub-btrfsd --syslog /.snapshots
```

with:

```
ExecStart=/usr/bin/grub-btrfsd --syslog -t
```

Regenerate GRUB config:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 15. Zram Setup

> **zram vs zswap:** `zram` compresses data entirely in RAM, reducing SSD wear. `zswap` uses a compressed RAM cache backed by a disk swap partition. This guide uses `zram`. If you regularly run out of RAM, `zswap` may suit you better.

Create the configuration:

```bash
mkdir -p /etc/systemd/zram-generator.conf.d/
nvim /etc/systemd/zram-generator.conf.d/zram.conf
```

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
```

Apply and reboot:

```bash
systemctl daemon-reexec
systemctl start /dev/zram0
reboot
```

---

## 16. Display Manager

If `ly` isn't already installed and enabled:

```bash
sudo pacman -S ly
sudo systemctl enable --now ly
```

---

## 17. Desktop Environment (COSMIC)

```bash
sudo pacman -S cosmic
```

---

## 18. AMD GPU Drivers

Enable the **multilib** repository (required for 32-bit libraries):

```bash
nvim /etc/pacman.conf
```

Uncomment both lines:

```ini
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Sync and install AMD drivers:

```bash
pacman -Sy
pacman -S mesa vulkan-radeon libva-mesa-driver lib32-mesa lib32-vulkan-radeon
```

This covers OpenGL, Vulkan, and 32-bit compatibility for Steam and Wine.
