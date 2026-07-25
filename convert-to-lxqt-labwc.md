# Converting Weston/Niri to LXQt with Labwc

This guide documents the installation, configuration, and troubleshooting of LXQt (Qt 6) with **Labwc** (an Openbox-inspired Wayland compositor based on wlroots) in the Android terminal virtualized environment.

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

> [!CAUTION]
> `sddm` can remain installed if desired, but `sddm.service` **must be stopped and masked** to prevent boot-time DRM seat conflicts and VM panics.

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

## 3. Create the Labwc + LXQt User Service & Socket

### `/etc/systemd/user/labwc.service`

Virtio-GPU acceleration under Android guest VMs requires disabling atomic mode setting and hardware cursors in wlroots (`WLR_DRM_NO_ATOMIC=1`, `WLR_NO_HARDWARE_CURSORS=1`).

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
Environment="WLR_DRM_NO_ATOMIC=1"
Environment="WLR_NO_HARDWARE_CURSORS=1"
Environment="WLR_RENDERER_ALLOW_SOFTWARE=1"
EnvironmentFile=-/home/droid/labwc.env
EnvironmentFile=-/home/droid/weston.env

ExecStart=/usr/bin/labwc -S lxqt-session
Restart=on-failure
RestartSec=1s

[Install]
WantedBy=graphical-session.target
```

### `/etc/systemd/user/labwc.socket`

```ini
[Unit]
Description=Labwc Wayland Compositor Socket
Documentation=man:labwc(1)

[Socket]
ListenStream=%t/wayland-0

[Install]
WantedBy=sockets.target
```

---

## 4. Update Display Startup Scripts

Update `/usr/local/bin/enable_display` and `/usr/local/bin/enable_gfxstream` to inject LXQt environment variables into systemd user manager and trigger `labwc` service start:

### `/usr/local/bin/enable_display`

```bash
#!/bin/bash
cat << EOF2 > /home/droid/labwc.env
WLR_DRM_NO_ATOMIC=1
WLR_NO_HARDWARE_CURSORS=1
WLR_RENDERER_ALLOW_SOFTWARE=1
XDG_CURRENT_DESKTOP=LXQt
XDG_SESSION_TYPE=wayland
XDG_SESSION_DESKTOP=LXQt
EOF2
cp /home/droid/labwc.env /home/droid/weston.env
sudo systemd-run --quiet --collect -E XDG_SESSION_TYPE=wayland --uid=1000 -p PAMName=login -p TTYPath=/dev/tty1 sleep 1d
systemctl --user set-environment WLR_DRM_NO_ATOMIC=1 WLR_NO_HARDWARE_CURSORS=1 WLR_RENDERER_ALLOW_SOFTWARE=1 XDG_CURRENT_DESKTOP=LXQt XDG_SESSION_TYPE=wayland XDG_SESSION_DESKTOP=LXQt
(sleep 3s; systemctl --user start labwc)& disown
export DISPLAY=:0
export MESA_LOADER_DRIVER_OVERRIDE=zink

export LIBGL_ALWAYS_SOFTWARE=1
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.json
export XDG_CURRENT_DESKTOP=LXQt
export XDG_SESSION_TYPE=wayland
export WLR_DRM_NO_ATOMIC=1
export WLR_NO_HARDWARE_CURSORS=1
```

### `/usr/local/bin/enable_gfxstream`

```bash
#!/bin/bash
cat << EOF2 > /home/droid/labwc.env
MESA_LOADER_DRIVER_OVERRIDE=zink
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json
MESA_VK_WSI_DEBUG=sw,linear
XWAYLAND_NO_GLAMOR=1
WLR_DRM_NO_ATOMIC=1
WLR_NO_HARDWARE_CURSORS=1
WLR_RENDERER_ALLOW_SOFTWARE=1
XDG_CURRENT_DESKTOP=LXQt
XDG_SESSION_TYPE=wayland
XDG_SESSION_DESKTOP=LXQt
EOF2
cp /home/droid/labwc.env /home/droid/weston.env
sudo systemd-run --quiet --collect -E XDG_SESSION_TYPE=wayland --uid=1000 -p PAMName=login -p TTYPath=/dev/tty1 sleep 1d
systemctl --user set-environment MESA_LOADER_DRIVER_OVERRIDE=zink VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json MESA_VK_WSI_DEBUG=sw,linear XWAYLAND_NO_GLAMOR=1 WLR_DRM_NO_ATOMIC=1 WLR_NO_HARDWARE_CURSORS=1 WLR_RENDERER_ALLOW_SOFTWARE=1 XDG_CURRENT_DESKTOP=LXQt XDG_SESSION_TYPE=wayland XDG_SESSION_DESKTOP=LXQt
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
export WLR_DRM_NO_ATOMIC=1
export WLR_NO_HARDWARE_CURSORS=1
```

---

## 5. Troubleshooting & Architectural Notes

### Virtio-GPU DRM Atomic Commit Errors
When running `wlroots`-based compositors (such as `labwc`) in virtio-gpu virtual machines, the following error may repeat in `journalctl`:
```text
labwc: [ERROR] [backend/drm/atomic.c:79] connector Virtual-1: Atomic commit failed: Invalid argument
```
**Solution:** `wlroots` requires explicit environment overrides to bypass DRM atomic commits and hardware mouse cursor plane commits on virtual DRM connectors:
- `WLR_DRM_NO_ATOMIC=1` — Enforces legacy DRM modesetting on `Virtual-1`.
- `WLR_NO_HARDWARE_CURSORS=1` — Uses software cursor rendering.
- `WLR_RENDERER_ALLOW_SOFTWARE=1` — Enables software rendering fallback.

### Wayland Socket Timing with Android Guest Agent (`linux_vm_manager`)
If `linux_vm_manager` attempts to bridge the display before `labwc` initializes `/run/user/1000/wayland-0`, the following error occurs:
```text
linux_vm_manager: Failed to connect to a Wayland server: No such file or directory
```
**Solution:** Ensure `/etc/systemd/user/labwc.socket` is installed and run `systemctl --user daemon-reload`. Socket activation keeps `/run/user/1000/wayland-0` ready for `linux_vm_manager` upon session startup.

---

## 6. Verification

Check the active session state:

```bash
systemctl --user status labwc.service
```

You should see `labwc` active and running `lxqt-session`, `pcmanfm-qt --desktop`, `lxqt-panel`, `lxqt-runner`, `lxqt-notificationd`, and `Xwayland`.
