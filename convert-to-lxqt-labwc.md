# Converting Weston/Niri to LXQt with Labwc

This guide documents the installation and configuration of LXQt (Qt 6) with **Labwc** (an Openbox-inspired Wayland compositor based on wlroots) in the Android terminal virtualized environment.

---

## Overview

The Android terminal environment runs inside a guest VM with a host-dependent display channel (`virtio-gpu` / `gfxstream`). Standard display managers (such as SDDM or Greetd) auto-starting at boot conflict with DRM seat allocation when the Android display activity is inactive.

To ensure stability and seamless display rendering, LXQt with Labwc is initialized on demand via `systemd-run` and a user-level systemd service (`labwc.service`) when an interactive terminal session opens.

---

## 1. Package Installation

Install LXQt desktop and Labwc compositor:

```bash
sudo apt install -y lxqt labwc
```

> [!NOTE]
> `sddm` can remain installed if desired, but `sddm.service` **must be stopped and masked** to prevent boot-time DRM seat conflicts.

```bash
sudo systemctl stop sddm.service
sudo systemctl disable sddm.service
sudo systemctl mask sddm.service
```

---

## 2. Stop and Mask Weston

Ensure Weston user service and socket are disabled and masked:

```bash
systemctl --user stop weston.service weston.socket
systemctl --user mask weston.service weston.socket
```

---

## 3. Create the Labwc + LXQt User Service

Create `/etc/systemd/user/labwc.service`:

```ini
[Unit]
Description=Labwc Wayland Compositor with LXQt Session
Documentation=man:labwc(1)
Before=graphical-session.target

[Service]
Type=simple
StandardOutput=journal
StandardError=journal
Environment="XDG_CURRENT_DESKTOP=LXQt"
Environment="XDG_SESSION_TYPE=wayland"
Environment="XDG_SESSION_DESKTOP=LXQt"
EnvironmentFile=-/home/droid/labwc.env
EnvironmentFile=-/home/droid/weston.env

ExecStart=/usr/bin/labwc -S lxqt-session
Restart=on-failure
RestartSec=1s

[Install]
WantedBy=graphical-session.target
```

---

## 4. Update Display Startup Scripts

Update `/usr/local/bin/enable_display` and `/usr/local/bin/enable_gfxstream` to inject LXQt environment variables into systemd user manager and trigger `labwc` service start:

### `/usr/local/bin/enable_display`

```bash
#!/bin/bash
echo > /home/droid/labwc.env
echo > /home/droid/weston.env
sudo systemd-run --quiet --collect -E XDG_SESSION_TYPE=wayland --uid=1000 -p PAMName=login -p TTYPath=/dev/tty1 sleep 1d
systemctl --user set-environment XDG_CURRENT_DESKTOP=LXQt XDG_SESSION_TYPE=wayland XDG_SESSION_DESKTOP=LXQt
(sleep 3s; systemctl --user start labwc)& disown
export DISPLAY=:0
export MESA_LOADER_DRIVER_OVERRIDE=zink

export LIBGL_ALWAYS_SOFTWARE=1
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.json
export XDG_CURRENT_DESKTOP=LXQt
export XDG_SESSION_TYPE=wayland
```

### `/usr/local/bin/enable_gfxstream`

```bash
#!/bin/bash
cat << EOF2 > /home/droid/labwc.env
MESA_LOADER_DRIVER_OVERRIDE=zink
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json
MESA_VK_WSI_DEBUG=sw,linear
XWAYLAND_NO_GLAMOR=1
XDG_CURRENT_DESKTOP=LXQt
XDG_SESSION_TYPE=wayland
EOF2
cp /home/droid/labwc.env /home/droid/weston.env
sudo systemd-run --quiet --collect -E XDG_SESSION_TYPE=wayland --uid=1000 -p PAMName=login -p TTYPath=/dev/tty1 sleep 1d
systemctl --user set-environment MESA_LOADER_DRIVER_OVERRIDE=zink VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json MESA_VK_WSI_DEBUG=sw,linear XWAYLAND_NO_GLAMOR=1 XDG_CURRENT_DESKTOP=LXQt XDG_SESSION_TYPE=wayland XDG_SESSION_DESKTOP=LXQt
(sleep 3s; systemctl --user start labwc)& disown
export DISPLAY=:0
export MESA_LOADER_DRIVER_OVERRIDE=zink

unset LIBGL_ALWAYS_SOFTWARE

export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json

# remove them after TILING_OPTIMAL-based swap chain is used.
export MESA_VK_WSI_DEBUG=sw,linear
export XWAYLAND_NO_GLAMOR=1
export LIBGL_KOPPER_DRI2=1
export XDG_CURRENT_DESKTOP=LXQt
export XDG_SESSION_TYPE=wayland
```

---

## 5. Verification

Check the active session state:

```bash
systemctl --user status labwc.service
```

You should see `labwc` active and running `lxqt-session`, `pcmanfm-qt --desktop`, `lxqt-panel`, `lxqt-runner`, `lxqt-notificationd`, and `Xwayland`.
