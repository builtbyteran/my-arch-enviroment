# my-arch-environment

Will contain the build process of my machines, including manual installation, configurations, and dependencies.

> I am consolidating 4 Arch guides into 1 for my own computers. I will document all 4 guides and choose the best options for my own use case from each.

---

## Goal

The goal of this Arch Linux install is to build a safe and capable machine for development, whether for learning to code or freelancing.

---

## Pre-Installation

Before booting, **disable Secure Boot** in your BIOS/UEFI settings, then boot into your Arch ISO via USB.

---

## Initial Setup

### Increase Terminal Font Size

```bash
setfont ter-132b
```

### Verify UEFI Mode

```bash
cat /sys/firmware/efi/fw_platform_size
```

> Should output `64` for 64-bit UEFI. If it outputs `32` or the file doesn't exist, you may be booted in BIOS/legacy mode.

### Check Network Interfaces

```bash
ip link
```

> `wlan0` will show as `DOWN` initially. It will show as `UP` once connected.

---

## Connect to Wi-Fi

Enter the `iwd` wireless daemon:

```bash
iwctl
```

Then run the following inside the `iwd` prompt — replace `name` with your device name (e.g. `wlan0`) and `SSID` with your network name:

iwd: device list
iwd: station name scan
iwd: station name get-networks
iwd: station name connect SSID

> `station name scan` produces no output, but it initiates a background network scan.

Exit `iwctl`, then verify the connection:

```bash
ping ping.archlinux.org
```

Press `Ctrl+C` to stop. Confirm your interface is now up:

```bash
ip link
```

# Arch Linux Installation Guide

> Based on Learn Linux TV (May 28th, 2026) — The Manual Method

---

## Table of Contents

1. [Partitioning the Drive](#1-partitioning-the-drive)
2. [Formatting Partitions](#2-formatting-partitions)
3. [Encrypting and Setting Up LVM](#3-encrypting-and-setting-up-lvm)
4. [Mounting](#4-mounting)
5. [Installing Base Packages](#5-installing-base-packages)
6. [Entering the Chroot Environment](#6-entering-the-chroot-environment)
7. [Kernels](#7-kernels)
8. [GPU Drivers](#8-gpu-drivers)
9. [Configuring mkinitcpio](#9-configuring-mkinitcpio)
10. [Locale Setup](#10-locale-setup)
11. [GRUB Setup](#11-grub-setup)
12. [Enabling Services and Rebooting](#12-enabling-services-and-rebooting)

---

## 1. Partitioning the Drive

```bash
fdisk /dev/nvme0n1
```

> **Note:** If prompted to remove an existing signature, press `Y`.

Inside `fdisk`, run the following commands:

| Command | Action                           |
| ------- | -------------------------------- |
| `g`     | Create a new GPT partition table |
| `n`     | New partition                    |
| `w`     | Write changes and exit           |

### Partition Layout

**Create the partition layout**

Command: g

**EFI Partition**

Command: n
Partition number: 1 (or default)
First sector: (default)
Last sector: +1G

**Boot Partition**

Command: n
Partition number: 2 (or default)
First sector: (default)
Last sector: +1G

**Linux Partition**

Command: n
Partition number: 3 (or default)
First sector: (default)
Last sector: (default — uses remaining space)

**Write the partition table:**

Command: w

> ⚠️ **Warning:** Writing immediately wipes the drive and applies your new partition layout.

---

## 2. Formatting Partitions

Format the EFI partition as FAT32:

```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

Format the boot partition as ext4:

```bash
mkfs.ext4 /dev/nvme0n1p2
```

> **Note:** Do **not** format the Linux partition yet if you plan to encrypt it. See the next section.

---

## 3. Encrypting and Setting Up LVM

> Skip this section if you do not want full-disk encryption.

### Encrypt the Linux Partition

```bash
cryptsetup luksFormat /dev/nvme0n1p3
```

When prompted:

- Type `YES` (uppercase) to confirm
- Enter and verify a passphrase — **do not forget it, or your data will be permanently inaccessible**

### Open the Encrypted Volume

```bash
cryptsetup open --type luks /dev/nvme0n1p3 lvm
```

Verify it opened successfully:

```bash
ls -l /dev/mapper
```

### Set Up LVM

```bash
pvcreate /dev/mapper/lvm
vgcreate volgroup0 /dev/mapper/lvm
lvcreate -L 30GB volgroup0 -n lv_root
lvcreate -L 400GB volgroup0 -n lv_home   # Adjust to your drive size; leave headroom
```

### Activate LVM

```bash
modprobe dm_mod
vgscan
vgchange -ay
```

### Format the Logical Volumes

```bash
mkfs.ext4 /dev/volgroup0/lv_root
mkfs.ext4 /dev/volgroup0/lv_home
```

---

## 4. Mounting

```bash
mount /dev/volgroup0/lv_root /mnt

mkdir /mnt/boot
mount /dev/nvme0n1p2 /mnt/boot

mkdir /mnt/home
mount /dev/volgroup0/lv_home /mnt/home
```

---

## 5. Installing Base Packages

```bash
pacstrap -i /mnt base
```

Accept the defaults and type `y` to proceed.

Generate the fstab:

```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
```

Verify it looks correct:

```bash
cat /mnt/etc/fstab
```

---

## 6. Entering the Chroot Environment

```bash
arch-chroot /mnt
```

### Set the Root Password

```bash
passwd
```

### Create a Local User

```bash
useradd -m -g users -G wheel yourusername
passwd yourusername
```

> Replace `yourusername` with your desired username.

### Install Essential Packages

```bash
pacman -S base-devel dosfstools grub efibootmgr gnome gnome-tweaks lvm2 mtools nano networkmanager os-prober sudo openssh
```

> - Replace `gnome gnome-tweaks` with your preferred desktop environment.
> - Replace `nano` with `vim` or another editor if preferred.
> - `openssh` is optional — only needed if you want to SSH into this machine.
> - If prompted to choose between `jack` and `pipewire-jack`, select **pipewire-jack**.

Enable SSH if installed:

```bash
systemctl enable sshd
```

---

## 7. Kernels

```bash
pacman -S linux linux-headers linux-lts linux-lts-headers
```

Accept the defaults.

---

## 8. GPU Drivers

Check which GPU you have:

```bash
lspci
```

Look for the **Display controller** entry.

**AMD:**

```bash
pacman -S mesa
# or for Vulkan support:
pacman -S vulkan-radeon lib32-vulkan-radeon
```

> For NVIDIA or Intel, substitute the appropriate driver packages.

---

## 9. Configuring mkinitcpio

```bash
nano /etc/mkinitcpio.conf
```

Find the line starting with `HOOKS=` (the uncommented one) and insert `sd-encrypt lvm2` between `block` and `filesystems`:

HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`), then rebuild the initramfs:

```bash
mkinitcpio -p linux
mkinitcpio -p linux-lts
```

---

## 10. Locale Setup

```bash
nano /etc/locale.gen
```

Uncomment your locale (e.g., `en_US.UTF-8 UTF-8`), then save and exit.

Generate the locale:

```bash
locale-gen
```

---

## 11. GRUB Setup

### Get the UUID of the Encrypted Partition

```bash
echo "rd.luks.name=$(blkid -s UUID -o value /dev/nvme0n1p3)=lvm"
```

Copy the output.

### Edit the GRUB Config

```bash
nano /etc/default/grub
```

Find `GRUB_CMDLINE_LINUX=""` and update it with the output from above, plus the root volume:

GRUB_CMDLINE_LINUX="rd.luks.name=<YOUR-UUID>=lvm root=/dev/volgroup0/lv_root"

Save and exit.

### Install GRUB

```bash
mkdir /boot/EFI
mount /dev/nvme0n1p1 /boot/EFI
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
```

### Copy Locale and Generate Config

```bash
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 12. Enabling Services and Rebooting

```bash
systemctl enable gdm
systemctl enable NetworkManager
exit
```

Unmount everything and reboot:

```bash
umount -a
reboot
```

---

🎉 **Congratulations! Arch Linux is installed.**
