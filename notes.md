# in order to install archi linux I have to connect to wifi using the iwctl command

At the [iwd]# prompt, run these in order:
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "YourSSID"
It asks for the passphrase. After that, type exit.

---

# Progress log (2026-10-02)

## Done: Phase 0, prep on the Mac
- Downloaded `archlinux-x86_64.iso` from a mirror linked on https://archlinux.org/download/
- Verified it: `shasum -a 256 ~/Downloads/archlinux-x86_64.iso` matched the hash on the site
  (`684ded26c63240ff4a41e8c25ee84ea6da233f557364821f13d12c2b0a9059a5`)
- balenaEtcher failed with `(0 , h.requestMetadata) is not a function` (an Etcher bug, not the ISO), so I used `dd`
- Flashed the USB (it was `disk5`, a 31 GB external drive):
  ```
  diskutil list external
  diskutil unmountDisk /dev/disk5
  sudo dd if="$HOME/Downloads/archlinux-x86_64.iso" of=/dev/rdisk5 bs=4m status=progress
  diskutil eject /dev/disk5
  ```
  - Lesson: in zsh `if=~/...` does NOT expand `~`. Use `"$HOME/..."`.
  - Confirmed the write: byte count equals the ISO size, and the hash of the first 1640497152 bytes of the stick matched the ISO.

## Done: Phase 1 so far (on the ASUS ROG laptop)
- BIOS: Secure Boot disabled. Boot priority didn't stick (an old OS came first), so I used the one-time boot menu (Esc / F8) to boot the USB
  - BIOS key: F2 (or Del). Boot menu: Esc (or F8). Not F5.
- Step 2: online via `iwctl` (see the top of this file). Verified with `ping -c 3 archlinux.org`
  - `scan` and `connect` print nothing when they work.
- Step 3: clock sync (`timedatectl set-ntp true`)
- Step 4: partitioning. `lsblk` showed:
  - `sda` 28.9G, removable = the USB installer (don't touch)
  - `nvme0n1` 476.9G = the internal disk, with 8 old partitions (Windows-style + Linux). I chose to **wipe everything**, which erased Windows and Ubuntu.
  - `cfdisk` only offered NTFS in its menu because the old partitions were still there, so I used `fdisk` instead:
    ```
    fdisk /dev/nvme0n1
    g            new GPT table
    n, Enter, Enter, +1G
    t, 1         type 1 = EFI System   (answer Y to "remove ntfs signature")
    n, Enter x3  rest of the disk
    w            write
    ```
  - Result (checked with `lsblk /dev/nvme0n1`):
    - `nvme0n1p1`  1G      EFI System
    - `nvme0n1p2`  475.9G  Linux filesystem

## Status when I stopped
- The partition table is saved on the disk and survives power-off.
- NOT sure whether I ran the format/mount commands (steps 5 and 6). Check with `lsblk -f /dev/nvme0n1`:
  - If FSTYPE is empty -> not formatted yet, start at step 5.
  - If it shows `vfat` and `ext4` -> formatted, only re-mount (step 6).
- The old OSes are gone, so the laptop can only boot the USB for now.

## When I come back: resume checklist
The live USB lives in RAM, so after a reboot I lose: Wi-Fi, the clock sync, and the mounts. The partitions stay.
1. Plug in the USB, power on, press Esc, pick the USB, choose "Arch Linux install medium"
2. Reconnect Wi-Fi (`iwctl`, see the top), test with `ping -c 3 archlinux.org`
3. `timedatectl set-ntp true`
4. `lsblk -f /dev/nvme0n1` to see the state, then continue below

## Next steps (from the plan, step 5 onwards)
5. Format (skip if `lsblk -f` already shows vfat + ext4)
   ```
   mkfs.fat -F32 /dev/nvme0n1p1
   mkfs.ext4 /dev/nvme0n1p2
   ```
6. Mount (root first, then EFI)
   ```
   mount /dev/nvme0n1p2 /mnt
   mount --mkdir /dev/nvme0n1p1 /mnt/boot
   ```
   Check with `lsblk /dev/nvme0n1`: `/mnt` next to p2, `/mnt/boot` next to p1.
7. Install the base system (`intel-ucode` or `amd-ucode`; check with `lscpu | grep "Model name"`)
   ```
   pacstrap -K /mnt base linux linux-firmware intel-ucode vim networkmanager sudo
   ```
8. `genfstab -U /mnt >> /mnt/etc/fstab`
9. `arch-chroot /mnt`
10. Time zone: `ln -sf /usr/share/zoneinfo/Europe/Bucharest /etc/localtime && hwclock --systohc`
11. Locale: uncomment `en_US.UTF-8` in `/etc/locale.gen`, `locale-gen`, `echo "LANG=en_US.UTF-8" > /etc/locale.conf`
12. Hostname: `echo "archlab" > /etc/hostname`
13. Root password: `passwd`
14. User: `useradd -m -G wheel amaimon && passwd amaimon`, then `EDITOR=vim visudo` and uncomment `%wheel ALL=(ALL:ALL) ALL`
15. `systemctl enable NetworkManager`
16. Bootloader (install the packages first, or it won't boot):
    ```
    pacman -S grub efibootmgr
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
    If the firmware ignores the boot entry: add `--removable` to `grub-install`. Inspect with `efibootmgr -v`.
17. `exit`, `umount -R /mnt`, `reboot`, **remove the USB**
18. Log in as my user, connect Wi-Fi with `nmtui`

If it won't boot: boot the USB again, `mount` the partitions, `arch-chroot /mnt`, and fix it. That's part of the lesson.

---

# Session 2 update (2026-10-02)

## Done
- Step 5: ran `mkfs.ext4` (the "discarding device blocks" message is normal TRIM on an SSD, just wait).
- Step 6: mounted, but **made a mistake**: the second mount used `nvme0n1p2` instead of `nvme0n1p1`
  - Wrong: `mount --mkdir /dev/nvme0n1p2 /mnt/boot`
  - Right:  `mount --mkdir /dev/nvme0n1p1 /mnt/boot`
  - `lsblk` showed it: `p2` had both `/mnt/boot` and `/mnt`, and `p1` had no mountpoint.
  - Fix: `umount /mnt/boot` then `mount /dev/nvme0n1p1 /mnt/boot`

## Status when I stopped
- Not yet confirmed the fix, and not sure `p1` was formatted (it needs `mkfs.fat -F32`).
- Check with `lsblk -f /dev/nvme0n1`. Goal:
  - `p1`: `vfat`, mounted at `/mnt/boot`
  - `p2`: `ext4`, mounted at `/mnt`
- If `p1` has no FSTYPE:
  ```
  umount /mnt/boot
  mkfs.fat -F32 /dev/nvme0n1p1
  mount /dev/nvme0n1p1 /mnt/boot
  ```
- After a reboot the mounts, Wi-Fi and clock sync are gone (the live USB runs from RAM). Redo: Wi-Fi (`iwctl`), `ping -c 3 archlinux.org`, `timedatectl set-ntp true`, then `lsblk -f /dev/nvme0n1` and fix the mounts as above.

## Next time: continue at step 7 (`pacstrap`)
Only after `lsblk -f` looks right:
```
pacstrap -K /mnt base linux linux-firmware intel-ucode vim networkmanager sudo
```
(`amd-ucode` if the CPU is AMD; check with `lscpu | grep "Model name"`.) Don't interrupt it. Then steps 8-18 from the list above.

---

# Session 3 update (2026-10-02): Arch is installed and boots

## Done: Phase 1 (manual install) is finished
- Step 6 fixed: `p1` (EFI, vfat) mounted at `/mnt/boot`, `p2` (ext4) mounted at `/mnt`. Verified with `lsblk -f` and `lsblk`.
- Step 7: `pacstrap -K /mnt base linux linux-firmware intel-ucode vim networkmanager sudo`
  - CPU is an Intel i7-9750H, so `intel-ucode`.
- Step 8: `genfstab -U /mnt >> /mnt/etc/fstab`
- Step 9: `arch-chroot /mnt`
- Steps 10-13 inside the chroot:
  - time zone Europe/Bucharest + `hwclock --systohc`
  - locale: my `sed` had a typo ("unterminated 's' command"), so I appended the line instead:
    ```
    echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
    locale-gen
    echo "LANG=en_US.UTF-8" > /etc/locale.conf
    ```
  - hostname `archlab`, root password set
- Step 14: user `amaimon` in group `wheel`, sudo enabled via `visudo` (`%wheel` line uncommented)
- Step 15: `systemctl enable NetworkManager`
- Step 16: `pacman -S grub efibootmgr`, `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`, `grub-mkconfig -o /boot/grub/grub.cfg` (no errors)
- Step 17: `exit`, `umount -R /mnt`, `reboot`, removed the USB
- The laptop booted into Arch (GRUB menu timed out and auto-booted "Arch Linux") and I logged in as `amaimon`.

## Wi-Fi on the installed system (UPDATE: it connected in the end, the notes below are only for reference if it drops again)
- `nmtui` -> "Activate a connection". I only see the 2.4 GHz network (the 5 GHz one is not listed).
- Selecting it gave: **"could not activate connection: the access point was not in the list"** (probably a stale scan).
- Things to try:
  ```
  nmcli device wifi rescan
  nmcli device wifi list
  nmcli device wifi connect "YourSSID" --ask
  ping -c 3 archlinux.org
  ```
  - Blocked or off? `rfkill list`, `nmcli radio wifi`, `sudo rfkill unblock all`, `nmcli radio wifi on`
  - Adapter detected? `ip link` (look for `wlan0` / `wlp...`)
  - Router on WPA3-only? Try WPA2 on the 2.4 GHz network.
  - Fallback: phone USB tethering or an Ethernet cable.
- 5 GHz missing is probably the regulatory domain: `sudo pacman -S wireless-regdb`, then uncomment `WIRELESS_REGDOM="RO"` in `/etc/conf.d/wireless-regdom` and reboot. Do this after getting online.

## Still to do
1. ~~Get online~~ DONE: Wi-Fi connected afterwards. Re-test with `ping -c 3 archlinux.org` after each boot.
2. **Phase 2: make it usable**
   - [ ] `sudo pacman -Syu`
   - [ ] `sudo pacman -S git base-devel man-db man-pages tldr htop curl wget openssh`
   - [ ] Check the GPU: `lspci | grep -i -E "vga|3d"` (NVIDIA on a ROG laptop needs extra drivers, which affects the desktop choice)
   - [ ] Pick a desktop: KDE Plasma or GNOME (+ `sddm` / `gdm`), or a tiling WM (Hyprland, Sway, i3)
   - [ ] AUR helper (`yay` or `paru`) and understand why the AUR is riskier
   - [ ] Shell setup (bash or zsh), aliases, prompt
   - [ ] Rule: do everyday tasks in the terminal
3. **Phase 3 onwards:** follow `linux-learning-plan.md` (fundamentals, Bandit, systemd, SSH, firewall, scripting, home server).
4. **Break-it-on-purpose exercises** from the plan, once things are stable.

## Handy reminders
- The command is `umount` (no "n").
- In zsh on the Mac, `if=~/...` does not expand `~`. Use `"$HOME/..."`.
- If the system won't boot: boot the USB, `mount` the partitions (`p2` -> `/mnt`, `p1` -> `/mnt/boot`), `arch-chroot /mnt`, and fix it. GRUB fallback: `grub-install --target=x86_64-efi --efi-directory=/boot --removable`.
- `nmtui` shows the same menu with arrow keys and Enter; `Esc` goes back.

---

# Session 3 (later): desktop install in progress

- My actual username is **`amaimon`** (the plan said `dragos`, but I created `amaimon`). Log in as `amaimon`.
- Wi-Fi works now (connected via `nmtui`).
- GPU check: `lspci` shows **Intel + NVIDIA** (hybrid graphics laptop). Plan: run the desktop on the Intel GPU only for now, and add NVIDIA later (optional: `nvidia-open` + `prime-run`).
- Chose **KDE Plasma** and ran:
  ```
  sudo pacman -S mesa vulkan-intel plasma-desktop plasma-nm sddm konsole dolphin
  ```
  - Provider choices during install: `pipewire-jack` (not `jack2`) and `qt6-multimedia-ffmpeg` (not the GStreamer one).
- Next: `sudo systemctl enable sddm` then `reboot`, and log in graphically.

## Still to do after the desktop works
- [ ] Log in at the SDDM screen as `amaimon` (session: Plasma Wayland, or X11 if there are problems)
- [ ] If it boots to a text login instead: `systemctl status sddm` and debug
- [ ] `sudo pacman -Syu` and the basics: `git base-devel man-db man-pages tldr htop curl wget openssh` (if not done yet)
- [ ] AUR helper (`yay` or `paru`), and understand why the AUR is riskier
- [ ] Shell setup (bash or zsh), aliases, prompt
- [ ] Wi-Fi 5 GHz: `wireless-regdb` + `WIRELESS_REGDOM="RO"` in `/etc/conf.d/wireless-regdom`
- [ ] Optional: NVIDIA drivers (`lspci | grep -i nvidia` to see the model)
- [ ] Continue with Phase 3 onwards in `linux-learning-plan.md`

---

# Session 3 (end): KDE Plasma works

## Done
- Plasma installed with `mesa vulkan-intel plasma-desktop plasma-nm sddm konsole dolphin`.
- `sudo systemctl enable sddm`, rebooted: graphical login works, desktop "looks amazing".
- Login user: `amaimon`. Desktop runs on the Intel GPU. NVIDIA drivers are still NOT installed (optional, later).

## Next session: customize the look and feel (anime / manga theme)

### Plan of attack (do it in this order, one layer at a time, so I can tell what broke)
1. **Pick one palette and stick to it.** Catppuccin is the obvious candidate because it has ports for almost everything below (Konsole, kitty, starship, KDE colors, GRUB, SDDM). Choose the flavor (Latte / Frappe / Macchiato / Mocha) and an accent color. Or take colors from a favorite anime's key art.
2. **Wallpaper** (static first, animated later).
3. **KDE global theme, icons, cursor** (System Settings -> Appearance -> "Get New ...").
4. **Terminal**: terminal emulator, prompt, `fastfetch` with an anime image/logo.
5. **Login screen (SDDM)**, then boot splash (Plymouth) and GRUB theme.
6. Only after that: animated wallpaper, a tiling WM (Hyprland/Sway) as a second session, etc.

### Ideas found in research (check each before installing)
- **KDE Global Themes** bundle plasma theme, colors, icons, cursor, splash and sometimes an SDDM theme.
  - System Settings -> Appearance -> Global Theme -> "Get New Global Themes". Search the KDE Store for "anime", "Catppuccin", etc.
  - Installing one may ask for my password: that's it installing the matching SDDM (login) theme.
  - Manual install paths: global themes `~/.local/share/plasma/look-and-feel/`, plasma themes `~/.local/share/plasma/desktoptheme/`, SDDM themes `/usr/share/sddm/themes` or `/etc/sddm/themes`-style locations, icons/cursors `~/.icons` or `~/.local/share/icons`.
  - The KDE Store has hundreds of SDDM themes, cursors and icon packs. One anime-style SDDM result to look at: ArcStarry-dark (https://store.kde.org/p/1644383).
- **Terminal (kitty or Konsole)**
  - kitty supports images, ligatures and lots of tweaks (https://itsfoss.com/kitty-customization/).
  - **Starship** prompt has a Catppuccin palette and a Powerline preset (https://github.com/catppuccin/starship).
  - **fastfetch** can show an image or ASCII art next to system info: an anime character as the logo. (Not covered in the search results: look up its config docs, and kitty/sixel image support.)
  - Konsole also takes color schemes and a background image, so I can start with Konsole and switch to kitty later.
  - A tool that searches wallpapers by tag (e.g. "anime") and previews them in the terminal: `ellibria-wall` (kitty/ghostty).
- **Boot splash (Plymouth)**: install `plymouth` (AUR per the source), add `plymouth` to the `HOOKS` in `/etc/mkinitcpio.conf`, add `quiet splash` to `GRUB_CMDLINE_LINUX_DEFAULT`, then `sudo plymouth-set-default-theme -R <theme>`. This one is riskier because it touches initramfs and GRUB. Back up first and keep the Arch USB.
  - Guide: https://www.debugpoint.com/install-plymouth-arch-linux/
- **Animated wallpaper**: Wallpaper Engine KDE plugin for Plasma 6 (needs Steam + Wallpaper Engine from the Workshop; AUR package; known limits: some scenes crash, no lock-screen support).
  - https://github.com/RainyPixel/wallpaper-engine-kde-plugin
  - Lighter alternatives: Shader Wallpaper and HTML Wallpaper plugins for Plasma 6.
- **Tiling WM, later**: Sway/Hyprland configs with "anime" themes exist on GitHub (e.g. https://www.mintlify.com/FraVelz/Config-Sway/reference/themes/anime). Fun, but it's a bigger project.
- **Not researched yet, worth a look**: GRUB themes (anime ones), fonts (a Nerd Font for the prompt icons, plus a Japanese font like Noto Sans CJK), Kvantum for Qt app theming, a `.face` login avatar, sound theme, Discord/Firefox/Spotify themes (Spicetify), and sourcing wallpapers.

### Rules for myself
- Change one thing, reboot or log out if needed, and write down what I did.
- Anything that touches boot (Plymouth, GRUB themes) -> do last, keep the USB handy.
- Prefer themes from the official KDE Store or well-known GitHub repos, and read the install script before running it (the same AUR-is-riskier lesson).
- Keep a dotfiles git repo for `~/.config` so I can undo or share the setup (good way to practice git, too).

## Also still to do (from the plan)
- [ ] `sudo pacman -Syu` and the basics: `git base-devel man-db man-pages tldr htop curl wget openssh`
- [ ] AUR helper (`yay` or `paru`) and understand why the AUR is riskier
- [ ] Shell setup (bash or zsh), aliases, prompt
- [ ] 5 GHz Wi-Fi: `wireless-regdb` + `WIRELESS_REGDOM="RO"` in `/etc/conf.d/wireless-regdom`
- [ ] Optional: NVIDIA drivers
- [ ] Phase 3 onwards in `linux-learning-plan.md`

## Sources used for the ideas above
- https://itsfoss.com/properly-theme-kde-plasma.md
- https://docs.kde.org/stable_kf6/en/plasma-workspace/kcontrol/lookandfeel/
- https://store.kde.org/p/1644383
- https://itsfoss.com/kitty-customization/
- https://github.com/catppuccin/starship
- https://www.debugpoint.com/install-plymouth-arch-linux/
- https://github.com/RainyPixel/wallpaper-engine-kde-plugin

---

# End of day 1 (2026-10-02): wrap-up

## Browser
- There is **no browser by default** (I installed `plasma-desktop`, the minimal package).
- Install one: `sudo pacman -S firefox` (alternatives: `chromium`, `qutebrowser`). Brave/Chrome are in the AUR, so wait for the AUR helper.

## Anime/manga theming: ideas beyond a wallpaper
(From my own knowledge, not all verified. Check each theme/tool before installing.)

**Whole-desktop theming**
- Global theme from the KDE Store (panel, colors, icons, cursor, window borders, splash in one go)
- Anime cursors and icon packs
- Custom login screen (SDDM), including a video background
- Boot splash (Plymouth) and GRUB theme (do last, riskiest)
- Custom startup / login / notification sounds

**Moving things**
- Animated wallpaper: Wallpaper Engine KDE plugin (needs Steam), or Shader/HTML Wallpaper plugins
- Desktop pet/mascot ("Shimeji" idea). Not confirmed which Linux versions still work.

**Terminal**
- Anime art with `fastfetch` or a kitty background image
- Random anime quote on login (my own script: good first scripting project)
- Starship prompt in my palette

**Automation (practice for the plan)**
- Wallpaper that changes by time of day or weather, using a script + a **systemd timer** (Phase 5)
- Colors generated from the wallpaper: Plasma accent-from-wallpaper, or `pywal`
- A dotfiles git repo for `~/.config` to restore/share the setup

**Home-server tie-ins (Phase 5)**
- Komga (self-hosted manga library + web reader), Jellyfin (own anime library)
- Avoiding tools that scrape unofficial streaming sites (legal/safety gray area)

**Bigger project**
- Hyprland or Sway as a second session: animations, blur, a fully hand-built "rice". Teaches the most Linux.

**Suggested order:** global theme + palette -> terminal -> login screen -> animated wallpaper -> boot splash -> desktop pet -> scripts.

## Decide before next session
- [ ] Which anime/manga is the theme? (drives the palette and wallpapers)
- [ ] Style: polished-and-cute, or dark-and-cyberpunk?
- [ ] Catppuccin flavor (Latte / Frappe / Macchiato / Mocha) + an accent color, or colors taken from key art

## Start of next session checklist
1. Log in as `amaimon`, open Konsole, check Wi-Fi: `ping -c 3 archlinux.org`
2. `sudo pacman -Syu`
3. `sudo pacman -S git base-devel man-db man-pages tldr htop curl wget openssh firefox`
4. AUR helper (`yay` or `paru`), then start theming in the order above
5. Keep the Arch USB somewhere handy, especially before touching Plymouth/GRUB

---

# Goal: learn to do this alone, by reinstalling Arch in a VM on the ROG

## Why / how
- Redo the install many times in a VM until I understand every step. Matches the last item in `linux-learning-plan.md`: "Reinstall Arch from scratch, aiming for under 30 minutes".
- Host: the ASUS ROG (x86_64, running Arch). The same ISO works, and KVM makes VMs fast. If the VM goes wrong, the real install stays safe.
- (A Mac as host would be worse: an Apple Silicon Mac can't run the x86_64 ISO natively.)

## VM setup checklist (figure out the details from the Arch Wiki)
1. Check hardware virtualization: `lscpu | grep -i virtualization` should show `VT-x`. If not, enable "Intel Virtualization Technology" in the BIOS (F2).
2. Read the Arch Wiki pages for **QEMU**, **libvirt** and **virt-manager**. Find out:
   - which packages to install (start from `qemu-full`, `libvirt`, `virt-manager`)
   - how to start and enable the `libvirtd` service
   - how to add my user to the `libvirt` group, and why I must log out and back in afterwards
3. Download the Arch ISO on the ROG and verify the checksum (`sha256sum`, same idea as `shasum -a 256` on the Mac).
4. Create the VM in virt-manager: ~4 GB RAM, 2-4 CPUs, 20-30 GB disk. **Use UEFI firmware (OVMF)**: without it `/sys/firmware/efi/efivars` doesn't exist and the `efibootmgr` / `grub-install --target=x86_64-efi` steps fail.
5. Take a **snapshot** of the empty VM before each run, so a reset takes seconds.

## Practice plan
- Run 1: only the Arch Wiki Installation guide (https://wiki.archlinux.org/title/Installation_guide), `notes.md` as a backup.
- Before each later run: close the notes, write the steps from memory, then compare. The gaps show what to study.
- Run 2: wiki only. Run 3+: nothing open, time myself.
- Then vary it each time: swap partition/swapfile, other filesystem (btrfs), other bootloader (systemd-boot), different desktop.
- After each run, write down: where I got stuck, why, and how I fixed it.

## Likely stumbling blocks
- UEFI not enabled in the VM -> no `efivars`.
- The disk is called `vda` in a VM, not `nvme0n1`, so partition names change (`vda1`, `vda2`).
- Forgetting to log out after joining the `libvirt` group.
- Typos I already made once: `~` doesn't expand after `if=` in zsh, the command is `umount` not `unmount`, and the second mount must use the EFI partition (`p1`), not root.

## Rules for getting help (ask Claude less)
1. Try first: read the full error, then check the wiki.
2. Ask for a hint ("which step should I look at?"), not the answer.
3. Use Claude as a reviewer: paste my finished commands and ask what's wrong. Don't ask what to type.

## Self-check: could I do each of these without help?
- [ ] Download and verify an ISO
- [ ] Boot the installer and get online (`iwctl`, `ping`)
- [ ] Partition (GPT: EFI + root), format, mount in the right order
- [ ] `pacstrap`, `genfstab`, `arch-chroot`
- [ ] Time zone, locale, hostname, root password, user + sudo
- [ ] Enable NetworkManager, install GRUB + `efibootmgr`, `grub-mkconfig`
- [ ] Unmount, reboot, remove the USB, log in, get online with `nmtui`
- [ ] Explain what each step does and why it is in that order
- [ ] Recover from a broken boot via the live USB + `arch-chroot`

---

# Session 4 (2026-10-05): first theming pass (Hatsune Miku)

## Done
- Theme choice: **Hatsune Miku**. Palette: Miku teal `#39C5BB` (accent), dark charcoal/navy base (`#1e1e2e` / `#16181d`), soft pink `#ff7eb6` for small highlights, off-white text `#e6f7f5`. Catppuccin Mocha + teal accent is the closest ready-made match.
- No dedicated Miku KDE theme turned up in a search of store.kde.org, so I built the look from parts.
- Steps I did: `sudo pacman -Syu`; fonts (`noto-fonts noto-fonts-cjk noto-fonts-emoji ttf-jetbrains-mono-nerd`); dark colors scheme + teal accent; a global theme from the KDE Store; a Miku wallpaper; Konsole color scheme + Nerd Font.
- Undo: switch the Global Theme back to **Breeze** (System Settings -> Appearance).
- Wallpaper rule: only use art I have the rights to / that the artist allows, and keep the credit.

## Next quick wins
- [ ] `fastfetch` with Miku art in the terminal
- [ ] Starship prompt in the Miku palette (teal + pink)
- [ ] SDDM login theme to match
- [ ] Cursor and icon pack in teal
- [ ] Animated wallpaper, then Plymouth/GRUB last (riskiest)
