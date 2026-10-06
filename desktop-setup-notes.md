# Desktop Setup Notes

Everything we set up on this KDE Plasma (Wayland) desktop, and how to change or redo it.

**System:** Arch Linux · KDE Plasma (Wayland) · Screen 1920×1080 · Terminal: Konsole · Shell: bash

---

## 1. Wallpaper (desktop + lock screen)

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

## 2. Minimizing Firefox (no minimize button)

**Quick ways:**
- **Meta+PgDown** minimizes the active window (KDE default; Meta is the Windows key)
- **Click Firefox's icon in the taskbar** when it's the active window
- **Alt+F3** opens the window menu → **Minimize**

**To get the button back:**
- **Firefox:** right-click an empty spot next to the tabs → **Customize Toolbar…** → tick **Title Bar** (bottom-left). Firefox then uses KDE's normal title bar.
- **KDE:** System Settings → Colors & Themes → Window Decorations → **Configure Titlebar Buttons…** → drag the minimize button onto the title bar.

---

## 3. Konsole profile: **MyselfAndI**

- Profile file: `~/.local/share/konsole/MyselfAndI.profile`
- Set as the **default profile** (in `~/.config/konsolerc` → `DefaultProfile=MyselfAndI.profile`)
- Uses the **Kaguya** color scheme (section 5)

To edit it: **☰ menu → Configure Konsole… → Profiles → MyselfAndI → Edit…**

Useful places in there:
- **Appearance:** color scheme, font, size, transparency and blur (Edit… on the scheme)
- **Scrolling:** scrollbar and history
- **Ctrl+Shift+M** hides or shows the menu bar

---

## 4. Kaguya picture in the terminal (fastfetch)

Every new terminal shows Kaguya's face on the left and system info on the right.

### Packages installed

```bash
sudo pacman -S fastfetch imagemagick
```

- **fastfetch** shows the system info and the picture
- **imagemagick** is needed so fastfetch can convert the picture for the terminal

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

## 5. Kaguya color scheme (red/black)

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

## File summary

| File | What it is |
|---|---|
| `~/Pictures/kaguya-wallpaper-1920x1080.jpg` | Fitted wallpaper (desktop + lock screen) |
| `~/.config/kscreenlockerrc` | Lock screen wallpaper setting |
| `~/.config/fastfetch/kaguya.png` | Face crop shown in the terminal |
| `~/.config/fastfetch/config.jsonc` | fastfetch settings |
| `~/.bashrc` | Runs fastfetch on new terminals |
| `~/.local/share/konsole/MyselfAndI.profile` | Konsole profile |
| `~/.local/share/konsole/Kaguya.colorscheme` | Red/black color scheme |
| `~/.config/konsolerc` | Makes MyselfAndI the default profile |
