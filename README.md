# my-arch-environment

Consolidated Arch Linux install guide for my personal machines — built for development, whether learning to code or freelancing.

> Based on: (YouTube Guides)
> Learn Linux TV (May 28th, 2026) - How to Install Arch Linux in 2026 - The Complete Step-by-Step Guide
> Josean Martinez (May 5th, 2026) - The Only Arch Linux Installation Guide You'll Ever Need
> DistroTube (Feb 10th, 2026) - An Arch Linux Installation Guide (2026)
> Bread on Penguins (Sep 25th, 2024) - Beginner friendly ARCH LINUX Installation Guide and Walkthrough

---

## Pre-Installation

Disable **Secure Boot** in BIOS/UEFI, then boot into the Arch ISO from USB.

```bash
setfont ter-132b        # increase terminal font size
timedatectl set-ntp true
```

**Verify UEFI mode** — should output `64`:

```bash
cat /sys/firmware/efi/fw_platform_size
```

---

## Connect to Wi-Fi

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

Exit, then verify:

```bash
ping -c 3 archlinux.org
```

---

## Update Mirrors

```bash
pacman -Sy reflector
reflector --country US --latest 5 --sort rate --save /etc/pacman.d/mirrorlist
```

---

## Table of Contents

1. [Partition the Drive](#1-partition-the-drive)
2. [Format Partitions](#2-format-partitions)
3. [Encrypt and Set Up Btrfs](#3-encrypt-and-set-up-btrfs)
4. [Mount](#4-mount)
5. [Install Base Packages](#5-install-base-packages)
6. [Enter Chroot](#6-enter-chroot)
7. [System Configuration](#7-system-configuration)
8. [ZRAM Setup](#8-zram-setup)
9. [GPU Drivers](#9-gpu-drivers)
10. [Configure mkinitcpio](#10-configure-mkinitcpio)
11. [GRUB Setup](#11-grub-setup)
12. [Install Desktop Environment](#12-install-desktop-environment)
13. [Enable Services and Reboot](#13-enable-services-and-reboot)

---

## 1. Partition the Drive

```bash
gdisk /dev/nvme0n1
```

> If prompted to remove an existing signature, press `Y`.

Delete all existing partitions by pressing `d` repeatedly, then create two new ones:

**EFI partition:**

```
n → 1 → (default) → +2G → EF00
```

**Linux partition** (LUKS container — rest of drive):

```
n → 2 → (default) → (default) → (default)
```

Write changes:

```
w
```

> ⚠️ This immediately wipes the drive.

---

## 2. Format Partitions

```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

> Do **not** format `/dev/nvme0n1p2` — it will be encrypted next.

---

## 3. Encrypt and Set Up Btrfs

> Skip this section if you don't want full-disk encryption.

```bash
cryptsetup luksFormat /dev/nvme0n1p2
```

Type `YES` (uppercase) and set a passphrase. **Don't lose it — your data will be permanently inaccessible without it.**

```bash
cryptsetup open /dev/nvme0n1p2 cryptroot
```

Format and create subvolumes:

```bash
mkfs.btrfs /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@var_log

umount /mnt
```

---

## 4. Mount

```bash
mount -o compress=zstd,subvol=@ /dev/mapper/cryptroot /mnt
mkdir -p /mnt/{home,.snapshots,var/log,boot}
mount -o subvol=@home /dev/mapper/cryptroot /mnt/home
mount -o subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
mount -o subvol=@var_log /dev/mapper/cryptroot /mnt/var/log
mount /dev/nvme0n1p1 /mnt/boot
```

---

## 5. Install Base Packages

```bash
pacstrap -K /mnt base linux linux-headers linux-lts linux-lts-headers linux-firmware networkmanager nano vim vi base-devel amd-ucode grub efibootmgr os-prober
```

Generate fstab:

```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab   # verify it looks correct
```

---

## 6. Enter Chroot

```bash
arch-chroot /mnt
```

---

## 7. System Configuration

### Timezone

```bash
ln -sf /usr/share/zoneinfo/America/Phoenix /etc/localtime
hwclock --systohc
```

> Find your timezone with `timedatectl list-timezones`. Note: `America/Phoenix` never observes DST.

### Locale

```bash
nano /etc/locale.gen
```

Uncomment `en_US.UTF-8 UTF-8` and `en_US ISO-8859-1`, save (`Ctrl+O`, `Enter`, `Ctrl+X`), then:

```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

### Hostname

```bash
echo "thinkpad" > /etc/hostname
```

### Root Password

```bash
passwd
```

### Create a User

```bash
useradd -m -g users -G wheel yourusername
passwd yourusername
```

### Configure sudo

```bash
EDITOR=nano visudo
```

Uncomment:

```
%wheel ALL=(ALL:ALL) ALL
```

### Essential Packages

```bash
pacman -S dosfstools efibootmgr mtools networkmanager sudo openssh git zram-generator
```

---

## 8. ZRAM Setup

```bash
nano /etc/systemd/zram-generator.conf
```

Add:

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
```

> Creates compressed swap in RAM using half your system RAM. No swap partition needed.

---

## 9. GPU Drivers

Check your GPU:

```bash
lspci | grep -i display
```

Enable the **multilib** repository (required for `lib32` packages):

```bash
nano /etc/pacman.conf
```

Uncomment both lines:

```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Then sync and install AMD drivers:

```bash
pacman -Sy
pacman -S mesa vulkan-radeon libva-mesa-driver lib32-mesa lib32-vulkan-radeon
```

> Covers OpenGL, Vulkan, and 32-bit compatibility (Steam/Wine).

---

## 10. Configure mkinitcpio

```bash
nano /etc/mkinitcpio.conf
```

Set the `HOOKS` line to:

```
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```

Rebuild the initramfs:

```bash
mkinitcpio -p linux
mkinitcpio -p linux-lts
```

---

## 11. GRUB Setup

Get the UUID of the encrypted partition:

```bash
blkid -s UUID -o value /dev/nvme0n1p2
```

Edit the GRUB config:

```bash
nano /etc/default/grub
```

Set `GRUB_CMDLINE_LINUX` using your UUID:

```
GRUB_CMDLINE_LINUX="rd.luks.name=<YOUR-UUID>=cryptroot root=/dev/mapper/cryptroot"
```

Install GRUB and generate the config:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub_uefi --recheck
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 12. Install Desktop Environment

```bash
pacman -S plasma kde-applications sddm sddm-kcm pipewire pipewire-alsa pipewire-pulse wireplumber
```

> - Wayland is built into `plasma` (KDE 6+) — no separate session package needed.
> - `kde-applications` is a large metapackage; swap in individual apps for a minimal install.
> - `pipewire` replaces PulseAudio.
> - If prompted, choose: gstreamer, pipewire-jack, noto-fonts, python-pyqt6, cronie, tessdata-eng.

---

## 13. Enable Services and Reboot

```bash
systemctl enable sddm
systemctl enable NetworkManager
exit

umount -R /mnt
reboot
```

> Remove the USB drive when the machine restarts. You'll be prompted for your LUKS passphrase on boot.

---

🎉 **Arch Linux is installed.**

On first login: **System Settings → Session** → confirm Wayland is selected. NetworkManager handles Wi-Fi from the KDE system tray.
