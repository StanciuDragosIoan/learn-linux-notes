# Customization guide: look and feel (anime theme)

Cosmetic changes only, done **after** a working system exists.
Base install and desktop: [`arch-install-guide.md`](arch-install-guide.md) (Parts 1 and 2). Nothing here is needed for a working OS.

**System:** Arch Linux · KDE Plasma (Wayland) · Screen 1920×1080 · Terminal: Konsole · Shell: bash
**Current theme:** *Kaguya-sama: Love Is War* (red / black).
**Earlier idea (not what is installed now):** Hatsune Miku, teal `#39C5BB` + charcoal base + pink `#ff7eb6`. Kept at the end as a ready palette in case I switch.

Rules: change one thing at a time, note what I changed, and put anything that touches the boot chain (Plymouth, GRUB theme) last, with the USB handy.

---

## 0. Packages used

```bash
sudo pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji ttf-jetbrains-mono-nerd
sudo pacman -S fastfetch imagemagick ffmpeg
```

- Fonts: `noto-fonts-cjk` gives Japanese characters; the Nerd Font is for prompt/terminal icons.
- `fastfetch` shows the system info and a picture; `imagemagick` lets it convert the picture for the terminal.
- `ffmpeg` is used to resize and crop the images (the Desktop notes below use it; check it is installed with `ffmpeg -version`).

## 1. Basic KDE look (System Settings)

1. Fonts: System Settings → Text & Fonts → Fonts (monospace: JetBrains Mono Nerd).
2. Colors: Appearance → Colors (a dark scheme, with an accent color from the theme).
3. Global theme: Appearance → Global Theme → **Get New…**. It may ask for my password to install the matching login-screen theme.
   Undo: switch back to **Breeze** in the same place.
4. Wallpaper: right-click the desktop → Configure Desktop and Wallpaper (or the command in section 2).
   Only use art I have the rights to / that the artist allows, and keep the credit.

---

## 2. Wallpaper (desktop + lock screen)

### The problem
The original picture was a small **728×800 portrait preview**, but the screen is **1920×1080 landscape**. To fill the screen, KDE had to enlarge it about 2.6× and cut off most of the top and bottom, so it looked zoomed in and blurry.

### The fix
Instead of cropping, the picture was **fitted to the screen height** and placed on the right. The empty space on the left was filled with the picture's own background red (`#df3744`), so there's no visible seam.

- Original (unchanged): `~/Pictures/kaguya-sama-love-is-war-...-preview.jpg`
- New wallpaper: `~/Pictures/kaguya-wallpaper-1920x1080.jpg`

Command used (ffmpeg):

```bash
ffmpeg -i kaguya-sama-*preview.jpg \
  -vf "scale=-2:1080:flags=lanczos,pad=1920:1080:1920-iw:0:color=0xdf3744" \
  -q:v 2 kaguya-wallpaper-1920x1080.jpg
```

- `scale=-2:1080` scales the picture to the screen height and keeps its proportions
- `pad=1920:1080:1920-iw:0` puts it on a 1920×1080 canvas, aligned to the right
- `color=0xdf3744` is the background red sampled from the picture

### Applying it

```bash
# Desktop
plasma-apply-wallpaperimage ~/Pictures/kaguya-wallpaper-1920x1080.jpg

# Lock screen
kwriteconfig6 --file kscreenlockerrc --group Greeter --group Wallpaper \
  --group org.kde.image --group General \
  --key Image "file:///home/amaimon/Pictures/kaguya-wallpaper-1920x1080.jpg"
```

Test the lock screen with **Meta+L**.

> **Tip:** the file name ends in "preview", so the site you got it from probably has a full-size version. Downloading that and running the same command gives a sharper result.

---

## 3. Lock-screen slideshow with clock

The lock screen cycles through every picture in `~/Pictures/Wallpapers/`, changing
every **60 seconds** in **random** order, with the clock always visible.

### The pictures
Downloaded from [Wallhaven](https://wallhaven.cc), safe-for-work only, chosen to match
the red/black theme:

```bash
# Search used (Wallhaven API)
curl "https://wallhaven.cc/api/v1/search?q=kaguya%20shinomiya&categories=010&purity=100&atleast=1920x1080&ratios=16x9&sorting=favorites"
```

- `purity=100` = safe-for-work only · `atleast=1920x1080` = full resolution · `ratios=16x9` = fits the screen
- Files: `wallhaven-136x99.jpg`, `rd12oj`, `8oyw82` (4K), `9mlwed`, `13y359`, `2e2v86`, `y86vy7`, `83krm2`, plus `kaguya-wallpaper-1920x1080.jpg`
- **To add more:** drop any 16:9 picture into `~/Pictures/Wallpapers/`. It is picked up automatically. Tall or small pictures can be fitted with the ffmpeg command from section 2.

### Settings (`~/.config/kscreenlockerrc`)

```bash
K="kwriteconfig6 --file kscreenlockerrc"
$K --group Greeter --key WallpaperPlugin org.kde.slideshow
G="--group Greeter --group Wallpaper --group org.kde.slideshow --group General"
$K $G --key SlidePaths /home/amaimon/Pictures/Wallpapers/
$K $G --key SlideInterval 60      # seconds between pictures
$K $G --key SlideshowMode 0       # 0 = random
$K $G --key FillMode 2            # fill the screen
$K --group Greeter --group LnF --group General --key alwaysShowClock true
```

- **Back to a single picture:** `kwriteconfig6 --file kscreenlockerrc --group Greeter --key WallpaperPlugin org.kde.image`
- **Using the GUI instead:** System Settings → Screen Locking → **Configure Appearance…**
- **When it locks:** System Settings → Screen Locking → "Lock screen automatically after"
- Test with **Meta+L**

---

## 4. Minimizing Firefox (no minimize button)

**Quick ways:**
- **Meta+PgDown** minimizes the active window (KDE default; Meta is the Windows key)
- **Click Firefox's icon in the taskbar** when it's the active window
- **Alt+F3** opens the window menu → **Minimize**

**To get the button back:**
- **Firefox:** right-click an empty spot next to the tabs → **Customize Toolbar…** → tick **Title Bar** (bottom-left). Firefox then uses KDE's normal title bar.
- **KDE:** System Settings → Colors & Themes → Window Decorations → **Configure Titlebar Buttons…** → drag the minimize button onto the title bar.

---

## 5. Konsole profile: **MyselfAndI**

- Profile file: `~/.local/share/konsole/MyselfAndI.profile`
- Set as the **default profile** (in `~/.config/konsolerc` → `DefaultProfile=MyselfAndI.profile`)
- Uses the **Kaguya** color scheme (section 7)

To edit it: **☰ menu → Configure Konsole… → Profiles → MyselfAndI → Edit…**

Useful places in there:
- **Appearance:** color scheme, font, size, transparency and blur (Edit… on the scheme)
- **Scrolling:** scrollbar and history
- **Ctrl+Shift+M** hides or shows the menu bar

---

## 6. Kaguya picture in the terminal (fastfetch)

Every new terminal shows Kaguya's face on the left and system info on the right.

### Packages
`fastfetch` and `imagemagick` (section 0).

### Files
- Picture (square face crop, 400×400): `~/.config/fastfetch/kaguya.png`
- Settings: `~/.config/fastfetch/config.jsonc`

How the face crop was made:

```bash
ffmpeg -i ~/Pictures/kaguya-sama-*preview.jpg \
  -vf "crop=600:600:128:20,scale=400:400:flags=lanczos" \
  ~/.config/fastfetch/kaguya.png
```

### fastfetch settings (`~/.config/fastfetch/config.jsonc`)

```jsonc
{
  "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
  "logo": {
    "type": "sixel",
    "source": "~/.config/fastfetch/kaguya.png",
    "width": 30,
    "height": 15,
    "padding": { "top": 1, "left": 2, "right": 3 }
  },
  "display": { "color": { "keys": "red" }, "separator": "  " },
  "modules": [
    "title", "separator", "os", "kernel", "uptime", "packages",
    "shell", "de", "terminal", "cpu", "gpu", "memory", "disk", "colors"
  ]
}
```

- **Picture size:** change `width` (columns) and `height` (rows)
- **Info shown:** add or remove entries in `modules`, e.g. `"battery"`, `"localip"`, `"weather"`
- **Label color:** `"keys": "red"`

### Running on every new terminal
Added at the end of `~/.bashrc`:

```bash
# Show Kaguya + system info on new terminals
[[ $- == *i* ]] && command -v fastfetch >/dev/null && fastfetch
```

To turn it off, delete or comment out (`#`) that line.

### Troubleshooting
- **`Gi=0;ENOTSUPPORTED`:** the logo type was `kitty-direct`, which tells the terminal to load the picture from a file. Konsole doesn't support that, so the fix was to use `"type": "sixel"`.
- **No picture at all:** check that `imagemagick` is installed. fastfetch needs it to convert the picture.

---

## 7. Kaguya color scheme (red/black)

- File: `~/.local/share/konsole/Kaguya.colorscheme`
- Used by the **MyselfAndI** profile (`ColorScheme=Kaguya`)

| Role | Normal | Bright |
|---|---|---|
| Background | `#120a0c` near black, red tint | `#1e1013` |
| Text | `#e8dcdc` soft off-white | `#ffffff` |
| Black | `#24181b` | `#5a4247` |
| **Red** | **`#df3744`** wallpaper red | `#ff5c68` |
| Green | `#8fb08a` muted | `#a8c9a2` |
| Yellow | `#d9a55b` gold | `#f0c27a` |
| Blue | `#6a7fa8` hair blue | `#8da0c7` |
| Magenta | `#c2507a` rose | `#e0789c` |
| Cyan | `#7fa9a8` snowflake teal | `#a3cccb` |
| White | `#d8cccc` | `#ffffff` |

To edit it: **☰ → Configure Konsole → Profiles → MyselfAndI → Edit → Appearance** → select **Kaguya** → **Edit…**. You can also add transparency there so the wallpaper shows through.

> If the Konsole settings window is open while the profile file is changed by hand, close it **without saving**, or it may overwrite the changes.

---

## 8. Redo order on a fresh install

1. Base system + Plasma: `arch-install-guide.md` Parts 1 and 2. Confirm it works **before** theming.
2. Section 0 (packages), then section 1 (fonts, colors, global theme).
3. Wallpaper (sections 2 and 3), then Konsole (sections 5 and 7), then fastfetch (section 6).
4. Back up `~/.config` and `~/.local/share/konsole` first (or keep them in a git dotfiles repo) so a redo takes minutes.

## 9. File summary

| File | What it is |
|---|---|
| `~/Pictures/kaguya-wallpaper-1920x1080.jpg` | Fitted wallpaper (desktop + lock screen) |
| `~/.config/kscreenlockerrc` | Lock screen wallpaper / slideshow / clock settings |
| `~/Pictures/Wallpapers/` | Pictures used by the lock-screen slideshow |
| `~/.config/fastfetch/kaguya.png` | Face crop shown in the terminal |
| `~/.config/fastfetch/config.jsonc` | fastfetch settings |
| `~/.bashrc` | Runs fastfetch on new terminals |
| `~/.local/share/konsole/MyselfAndI.profile` | Konsole profile |
| `~/.local/share/konsole/Kaguya.colorscheme` | Red/black color scheme |
| `~/.config/konsolerc` | Makes MyselfAndI the default profile |

---

## 10. Ideas for later (not done)

- [ ] Starship prompt in the theme colors
- [ ] SDDM login theme, matching cursor and icon pack
- [ ] Animated wallpaper (Wallpaper Engine KDE plugin needs Steam; Shader/HTML Wallpaper plugins for Plasma 6)
- [ ] Boot splash (Plymouth) and GRUB theme: last, riskiest, keep the USB handy
- [ ] Wallpaper rotation by time of day with a systemd timer
- [ ] Dotfiles git repo for `~/.config`
- [ ] Tiling WM (Hyprland/Sway) as a second session

## 11. Spare palette: Hatsune Miku (earlier idea)

- Miku teal `#39C5BB` (accent)
- Dark charcoal/navy base `#1e1e2e` or `#16181d`
- Soft pink `#ff7eb6` for small highlights
- Off-white text `#e6f7f5`
- Catppuccin Mocha with a teal accent is the closest ready-made match. No dedicated Miku KDE theme turned up in a search of the KDE Store.
