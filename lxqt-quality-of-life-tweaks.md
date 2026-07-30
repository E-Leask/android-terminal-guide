# LXQt Quality-of-Life (QoL) Tweaks & Enhancements

This document details post-installation quality-of-life tweaks and usability enhancements for running LXQt with Labwc in the Android terminal environment. These configurations are optional, but significantly improve UI scaling, visual polish, and workspace aesthetics.

---

## 1. Removing the Blank ~100x100 White Square (`xwaylandvideobridge`)

### Issue
When LXQt starts under Wayland, `xwaylandvideobridge` auto-starts to enable legacy X11 applications to capture screen buffers. It creates a 100x100 dummy window at coordinates `(0,0)`, which shows up as a blank white box in the top-left corner of the screen.

### Solution
Disable `xwaylandvideobridge` from autostarting by creating `~/.config/autostart/org.kde.xwaylandvideobridge.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=XWayland Video Bridge
Exec=xwaylandvideobridge
Hidden=true
X-GNOME-Autostart-enabled=false
```

---

## 2. High-DPI Display & Font Scaling

Android devices frequently use high-DPI displays where default 1.0 scaling makes text, icons, and window borders appear too small.

### A. Output Scaling via `wlr-randr`
Create `~/.config/labwc/autostart`:

```bash
#!/bin/sh
# Scale the virtual display output (1.25x or 1.5x based on preference)
wlr-randr --output Virtual-1 --scale 1.25
```

Make the autostart script executable:

```bash
chmod +x ~/.config/labwc/autostart
```

### B. Qt & Font DPI Configuration
Create `~/.config/labwc/environment` to pass high-DPI environment variables to all Qt/Wayland applications:

```bash
QT_ENABLE_HIGHDPI_SCALING=1
XFT_DPI=120
```

---

## 3. Desktop Icons & Application Shortcuts Setup

In LXQt, desktop icons and launchers are managed by `pcmanfm-qt --desktop` using the `~/Desktop` directory.

### A. Create `~/Desktop` Directory & Populate Shortcuts
Create the directory and copy desired `.desktop` application launchers:

```bash
mkdir -p ~/Desktop
cp /usr/share/applications/qterminal.desktop ~/Desktop/
cp /usr/share/applications/pcmanfm-qt.desktop ~/Desktop/
cp /usr/share/applications/chromium.desktop ~/Desktop/
chmod +x ~/Desktop/*.desktop
```

### B. Toggle System Icons (Home, Trash, Computer, Network)
System icons are configured in `~/.config/pcmanfm-qt/lxqt/settings.conf`:

```ini
[Desktop]
DesktopShortcuts=Home, Trash, Computer, Network
```

---

## 4. LXQt Panel & Icon Dimension Tuning

Adjust panel height and icon dimensions in `~/.config/lxqt/panel.conf` to improve touch targets and readability:

```ini
[panel1]
iconSize=28
panelSize=40
```

---

## 5. Desktop Wallpaper Management

Desktop wallpapers in LXQt are managed by `pcmanfm-qt`.

### A. CLI Command (Instant Set)
Set a new wallpaper from the command line:

```bash
pcmanfm-qt -w /path/to/your/wallpaper.jpg --wallpaper-mode zoom
```

Available modes: `zoom` (scale & crop), `fit` (preserve aspect ratio), `stretch`, `center`, `tile`.

### B. GUI Preference Window
1. Right-click any open space on the desktop canvas.
2. Select **Desktop Preferences**.
3. Under **Wallpaper**, select your image file and desired layout mode.
