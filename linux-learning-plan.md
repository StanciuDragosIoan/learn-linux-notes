# Linux Learning Plan: Arch on the Spare Laptop

Goal: get really good at Linux by using it daily, breaking it, and fixing it.
Pace: 30–60 min/day, consistently. Expect ~3 months to feel solid.

---

## Phase 0: Prep (today, ~30 min)

- [ ] Back up anything on the laptop you care about (the install wipes the disk)
- [ ] Download the Arch ISO: https://archlinux.org/download/
- [ ] Flash it to a USB stick (Ventoy, balenaEtcher, or `dd`)
- [ ] Open the Arch Wiki **Installation guide** on your phone or another machine: https://wiki.archlinux.org/title/Installation_guide
- [ ] Have a fallback for internet: Ethernet cable or phone USB tethering
- [ ] In BIOS/UEFI: disable Secure Boot and set the USB as the first boot device

---

## Phase 1: Manual Arch install (one evening, 2–4 h the first time)

Do it by hand, not with `archinstall`. The point is to learn what each step does.

1. **Boot the USB.** Confirm UEFI mode: `ls /sys/firmware/efi/efivars`
2. **Get online.**
   - Wi-Fi: `iwctl` → `station wlan0 connect "SSID"`
   - Test: `ping archlinux.org`
3. **Sync the clock:** `timedatectl set-ntp true`
4. **Partition the disk** (`lsblk` to find it, then `cfdisk /dev/nvme0n1` or `/dev/sda`):
   - 1 GB → EFI System
   - Rest → Linux filesystem
   - (Optional: a swap partition, or use a swapfile later)
5. **Format:**
   - `mkfs.fat -F32 /dev/<efi-partition>`
   - `mkfs.ext4 /dev/<root-partition>`
6. **Mount:**
   - `mount /dev/<root-partition> /mnt`
   - `mount --mkdir /dev/<efi-partition> /mnt/boot`
7. **Install the base system:**
   `pacstrap -K /mnt base linux linux-firmware intel-ucode` (or `amd-ucode`) `vim networkmanager sudo`
8. **Generate fstab:** `genfstab -U /mnt >> /mnt/etc/fstab`
9. **Chroot in:** `arch-chroot /mnt`
10. **Time zone:** `ln -sf /usr/share/zoneinfo/Europe/Bucharest /etc/localtime && hwclock --systohc`
11. **Locale:** uncomment `en_US.UTF-8` in `/etc/locale.gen`, run `locale-gen`, then `echo "LANG=en_US.UTF-8" > /etc/locale.conf`
12. **Hostname:** `echo "archlab" > /etc/hostname`
13. **Root password:** `passwd`
14. **Your user:**
    - `useradd -m -G wheel dragos && passwd dragos`
    - `EDITOR=vim visudo` → uncomment `%wheel ALL=(ALL:ALL) ALL`
15. **Networking at boot:** `systemctl enable NetworkManager`
16. **Bootloader (GRUB):**
    - `pacman -S grub efibootmgr`
    - `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`
    - `grub-mkconfig -o /boot/grub/grub.cfg`
17. **Reboot:** `exit`, `umount -R /mnt`, `reboot`, and remove the USB
18. Log in as your user and connect to Wi-Fi with `nmtui`

If it doesn't boot, that's part of the lesson. Boot the USB again, `mount` and `arch-chroot` back in, and fix it.

---

## Phase 2: Make it usable (week 1)

- [ ] Update the system: `sudo pacman -Syu`
- [ ] Install basics: `git base-devel man-db man-pages tldr htop curl wget openssh`
- [ ] Pick a desktop:
  - Easier: KDE Plasma or GNOME (+ `sddm` / `gdm`)
  - More learning: a tiling window manager (Hyprland, Sway, or i3) configured by hand
- [ ] Install an AUR helper (`yay` or `paru`) and understand what the AUR is and why it's riskier
- [ ] Set up your shell (bash or zsh), with aliases and a prompt
- [ ] Rule: **do everyday tasks in the terminal** (files, installs, disk usage with `df -h` / `du -sh`)

---

## Phase 3: Fundamentals (weeks 1–3)

Read _The Linux Command Line_ (William Shotts, free PDF: https://linuxcommand.org/tlcl.php) alongside this.

- [ ] Filesystem layout: `/etc`, `/var`, `/usr`, `/home`, `/proc`, `/dev`
- [ ] Navigation and files: `ls -la`, `cd`, `cp`, `mv`, `rm`, `ln -s`, `find`, `locate`
- [ ] Text: `cat`, `less`, `head`, `tail -f`, `grep`, `sed`, `awk`, `sort`, `uniq`, `wc`, `cut`
- [ ] Pipes and redirection: `|`, `>`, `>>`, `2>&1`, `tee`, `xargs`
- [ ] Permissions: `chmod` (symbolic + octal), `chown`, `umask`, sudo, groups
- [ ] Processes: `ps aux`, `top`/`htop`, `kill`, signals, `jobs`, `bg`, `fg`, `nohup`
- [ ] Package management: `pacman -S / -R / -Ss / -Qi / -Qdt`, cache cleanup
- [ ] Environment: `env`, `export`, `PATH`, `.bashrc` vs `.bash_profile`
- [ ] **OverTheWire Bandit**, levels 0 → ~20: https://overthewire.org/wargames/bandit/
- [ ] Habit: when an error appears, read it fully before searching

---

## Phase 4: Daily use (month 1)

- [ ] systemd: `systemctl status/start/enable`, write your own `.service` unit
- [ ] Logs: `journalctl -u <service>`, `journalctl -b`, `dmesg`
- [ ] SSH: key-based login from your main machine, disable password auth
- [ ] Firewall: `ufw` or `nftables`, only allow what you need
- [ ] Disks: `lsblk`, `mount`, `/etc/fstab`, add a swapfile
- [ ] Networking basics: `ip a`, `ip r`, `ss -tulpn`, `dig`, `curl -v`
- [ ] Users and groups: create, modify, and delete users, and understand `/etc/passwd` and `/etc/shadow`

---

## Phase 5: Scripting + home server (months 2–3)

- [ ] Bash scripting: variables, conditionals, loops, functions, exit codes, `set -euo pipefail`
- [ ] Write small, useful scripts:
  - [ ] Backup script (`rsync` + timestamped folders)
  - [ ] Bulk file renamer
  - [ ] System health report (disk, memory, failed services)
- [ ] Schedule them with **systemd timers** (and try cron to compare)
- [ ] Turn the laptop into a home server:
  - [ ] Docker on Arch, run a couple of services
  - [ ] Reverse proxy (Caddy or nginx)
  - [ ] Monitoring (e.g. `netdata` or Prometheus + node_exporter)
- [ ] Finish Bandit, then try **Natas** or **Leviathan** on OverTheWire

---

## Phase 6: Go deeper (month 3+)

- [ ] Read _How Linux Works_ (Brian Ward): boot process, kernel, devices, networking
- [ ] Understand the boot chain: UEFI → GRUB → kernel → initramfs → systemd
- [ ] Kernel modules: `lsmod`, `modprobe`
- [ ] Optional: build a kernel from source
- [ ] Optional security track: run Kali in a VM, try TryHackMe / HackTheBox beginner paths

---

## Break-it-on-purpose exercises

Do these on purpose, then recover. Take a snapshot or note what you changed first.

- [ ] Break `/etc/fstab` → boot fails → fix it from the USB with `arch-chroot`
- [ ] Remove your user from `wheel` → lose sudo → recover as root
- [ ] `chmod -x` a binary you use → figure out why it fails
- [ ] Disable NetworkManager → get back online manually
- [ ] Fill the disk with a big file → see what breaks → find it with `du`
- [ ] Delete the GRUB config → reinstall the bootloader
- [ ] Reinstall Arch from scratch, aiming for under 30 minutes

---

## When you get stuck

| Resource                               | Use it for                                   |
| -------------------------------------- | -------------------------------------------- |
| Arch Wiki — https://wiki.archlinux.org | First stop for almost anything               |
| `tldr <command>`                       | Quick practical examples                     |
| `man <command>`                        | Full reference                               |
| explainshell.com                       | Breaking down a command you don't understand |
| Unix & Linux Stack Exchange            | Specific problems with tested answers        |
| r/archlinux, r/linuxquestions          | Community help                               |

---

## Optional extras

- Linux Journey (free, structured): https://linuxjourney.com
- freeCodeCamp's Linux course (YouTube), for a guided walkthrough
- KodeKloud's free Linux courses, for browser labs

---

## Weekly check-in

- [ ] What did I break this week, and how did I fix it?
- [ ] Which command did I look up more than twice? Make an alias or note for it.
- [ ] Did I do at least 5 practice sessions?
