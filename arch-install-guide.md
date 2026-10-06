# Arch Linux on the ASUS ROG: install guide

Everything here was done by hand on an ASUS ROG laptop (Intel i7-9750H + NVIDIA, UEFI, NVMe disk).
The guide covers a working system only; no theming:

- **Part 1: Bare-bones install.** Gets you a working, bootable Arch with networking and a user.
- **Part 2: Working desktop.** KDE Plasma, a browser and the basic tools.
- **Part 3: Look and feel.** Lives in [`customization-guide.md`](customization-guide.md). Nothing there is needed for a working OS.

Reference for every step: https://wiki.archlinux.org/title/Installation_guide

Values to change for a new machine/VM: disk name (`nvme0n1`, in a VM usually `vda`), time zone, hostname, username, CPU microcode (`intel-ucode` or `amd-ucode`).

---

# Part 1: Bare-bones install (a working OS, no desktop)

## 1.0 Prep on another machine

1. Download `archlinux-x86_64.iso` from a mirror linked on https://archlinux.org/download/
   (the file in the mirror listing is a symlink to the latest dated ISO; the tiny size shown is normal).
2. Verify the checksum against the SHA256 on archlinux.org (take it from archlinux.org, not from the mirror):
   ```sh
   shasum -a 256 ~/Downloads/archlinux-x86_64.iso      # macOS
   sha256sum archlinux-x86_64.iso                       # Linux
   ```
3. Flash it to a USB stick. This erases the stick.
   - balenaEtcher (etcher.balena.io): Flash from file -> Select target -> Flash.
     (On my Mac it failed with `(0 , h.requestMetadata) is not a function`: an Etcher bug, not a bad ISO.)
   - Or `dd` on macOS:
     ```sh
     diskutil list external                 # find the stick, e.g. /dev/disk5; check the size!
     diskutil unmountDisk /dev/disk5
     sudo dd if="$HOME/Downloads/archlinux-x86_64.iso" of=/dev/rdisk5 bs=4m status=progress
     diskutil eject /dev/disk5
     ```
     In zsh, `if=~/...` does NOT expand `~`. Use `"$HOME/..."`.
   - Optional check that the stick is a byte-for-byte copy: hash the first N bytes of the device,
     where N is the ISO size (`stat -f%z <iso>`), and compare with the ISO hash.
4. Back up everything on the target machine. The install wipes the disk.
5. Have internet ready: Wi-Fi password, or Ethernet / phone USB tethering.
6. BIOS/UEFI on the ROG: **F2** (or Del) opens setup, **Esc** (or F8) opens the one-time boot menu. Not F5.
   - Disable **Secure Boot** (Security tab; if greyed out, set an admin password first).
   - Boot priority may not stick if another OS is installed. Use the one-time boot menu and pick the USB.

## 1.1 Boot the installer

- Pick the USB in the boot menu, then **Arch Linux install medium**.
- Check UEFI mode: `ls /sys/firmware/efi/efivars` (a list means UEFI; "No such file" means BIOS mode, which this guide doesn't cover).

## 1.2 Get online

Wi-Fi:
```sh
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "YourSSID"      # asks for the passphrase
exit
```
- `scan` and `connect` print nothing when they work.
- If the device is powered off: `device wlan0 set-property Powered on`, and `rfkill unblock all` outside iwctl.
- If there is no Wi-Fi device at all, use USB tethering or Ethernet.

Test, then sync the clock:
```sh
ping -c 3 archlinux.org
timedatectl set-ntp true
```

## 1.3 Partition (this wipes the disk)

Find the right disk. The internal disk is big and has the old partitions; the USB stick is small and shows `RM=1`:
```sh
lsblk
```
Partition with `fdisk` (clearer than `cfdisk` when old partitions exist):
```sh
fdisk /dev/nvme0n1
g            # new empty GPT table (discards all old partitions)
n            # new partition 1: Enter, Enter, then +1G
t, 1         # type 1 = EFI System   (answer Y to "remove the signature")
n            # new partition 2: Enter x3 (rest of the disk)
w            # write and exit
```
Check: `lsblk /dev/nvme0n1` shows `nvme0n1p1` (1G) and `nvme0n1p2` (the rest).
Nothing is erased until `w`.

## 1.4 Format and mount

```sh
mkfs.fat -F32 /dev/nvme0n1p1          # EFI partition
mkfs.ext4 /dev/nvme0n1p2              # root (the "discarding device blocks" message is normal TRIM on an SSD)
mount /dev/nvme0n1p2 /mnt             # root FIRST
mount --mkdir /dev/nvme0n1p1 /mnt/boot   # then EFI
lsblk /dev/nvme0n1                    # p2 on /mnt, p1 on /mnt/boot
```
Mistake I made once: mounting `p2` on `/mnt/boot`. The EFI partition (`p1`) goes on `/mnt/boot`.

## 1.5 Install the base system

```sh
pacstrap -K /mnt base linux linux-firmware intel-ucode vim networkmanager sudo
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```
Use `amd-ucode` on an AMD CPU (check with `lscpu | grep "Model name"`). Don't interrupt `pacstrap`.

## 1.6 Configure inside the chroot

```sh
ln -sf /usr/share/zoneinfo/Europe/Bucharest /etc/localtime
hwclock --systohc

echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

echo "archlab" > /etc/hostname
passwd                                   # root password

useradd -m -G wheel amaimon
passwd amaimon
EDITOR=vim visudo                        # uncomment:  %wheel ALL=(ALL:ALL) ALL

systemctl enable NetworkManager
```
Note: my first attempt used `sed` to uncomment the locale line and failed with "unterminated 's' command" (a typo); appending the line is simpler.

## 1.7 Bootloader (the step that makes it boot)

```sh
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```
You should see "Installation finished. No error reported" and a `Found linux image` line.

## 1.8 Reboot

```sh
exit
umount -R /mnt          # the command is "umount", not "unmount"
reboot                  # remove the USB as it restarts
```
GRUB shows a menu, auto-boots "Arch Linux", and you get a login prompt. Log in as your user.

Connect to Wi-Fi on the installed system:
```sh
nmtui          # Activate a connection -> pick the network -> password
ping -c 3 archlinux.org
```
If `nmtui` says the access point is not in the list, the scan was stale:
```sh
nmcli device wifi rescan
nmcli device wifi list
nmcli device wifi connect "YourSSID" --ask
```
Also check `rfkill list` and `nmcli radio wifi`.

**Part 1 is finished: this is a working, bootable Arch with networking.**

## If it won't boot (recovery)

Boot the USB again, then:
```sh
mount /dev/nvme0n1p2 /mnt
mount --mkdir /dev/nvme0n1p1 /mnt/boot
arch-chroot /mnt
```
Fix what is broken. If the firmware ignores the "GRUB" boot entry, install GRUB to the fallback path:
```sh
grub-install --target=x86_64-efi --efi-directory=/boot --removable
```
Inspect the UEFI entries with `efibootmgr -v`.

---

# Part 2: Working desktop (functional, still default looks)

Run as your normal user, online.

```sh
sudo pacman -Syu
sudo pacman -S git base-devel man-db man-pages tldr htop curl wget openssh firefox
```

Hybrid graphics (`lspci | grep -i -E "vga|3d"` shows Intel + NVIDIA): run the desktop on the Intel GPU first and add NVIDIA later (optional).

Install KDE Plasma:
```sh
sudo pacman -S mesa vulkan-intel plasma-desktop plasma-nm sddm konsole dolphin
sudo systemctl enable sddm
reboot
```
- When pacman asks for a provider, I picked `pipewire-jack` (not `jack2`) and `qt6-multimedia-ffmpeg` (not the GStreamer one).
- After the reboot, log in at the SDDM screen (session: Plasma Wayland, or X11 if there are problems).
- There is no browser by default; `firefox` is in the list above.
- If it boots to a text login instead of the graphical one: `systemctl status sddm`.

Optional, 5 GHz Wi-Fi networks missing: `sudo pacman -S wireless-regdb`, then uncomment `WIRELESS_REGDOM="RO"` in `/etc/conf.d/wireless-regdom` and reboot.

**Part 2 is finished: a usable KDE desktop with default looks.**

---

# Part 3: Look and feel

Moved to a separate file so the base install stays clean: see [`customization-guide.md`](customization-guide.md).

---

# Appendix: common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `if=~/file` in zsh | `dd: ~/...: No such file or directory` | use `"$HOME/file"` |
| Wrong disk in `dd`/`fdisk` | wipes the wrong drive | always check `diskutil list external` / `lsblk` sizes and `RM` first |
| `p2` mounted on `/mnt/boot` | `lsblk` shows both mounts on `p2` | `umount /mnt/boot`, mount `p1` there |
| `unmount` | command not found | it is `umount` |
| No `grub`/`efibootmgr` before `grub-install` | doesn't boot | `pacman -S grub efibootmgr` first |
| USB left in after install | installer boots again | remove the USB at reboot |
| VM without UEFI | no `/sys/firmware/efi/efivars` | enable OVMF firmware |
| VM disk name | `nvme0n1` doesn't exist | it is usually `vda` |
