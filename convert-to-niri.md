# Converting from Weston to Niri

This guide outlines the steps to migrate the initial graphical session from Weston to Niri (a scrollable-tiling Wayland compositor) within the Android terminal environment.

## Overview

The terminal environment relies on a combination of `systemd-run` and a user service to establish a Wayland session on a virtual terminal (`tty1`). By default, the `enable_display` and `enable_gfxstream` scripts launch `weston`. To use Niri, we simply need to update these scripts to invoke `niri.service` and adjust how environment variables are passed.

---

## 1. Stop and Mask Weston

First, ensure that Weston is stopped and masked so it doesn't conflict with Niri.

```bash
# Stop the current Weston service and socket
systemctl --user stop weston.service weston.socket

# Mask them to prevent accidental activation
systemctl --user mask weston.service weston.socket
```

## 2. Update the Display Scripts

The startup scripts located in `/usr/local/bin/` must be updated to launch Niri.

### Updating `enable_display`

Open `/usr/local/bin/enable_display` in your preferred editor (e.g., `nano` or `micro`) and find the line that starts Weston:

**Find:**
```bash
(sleep 3s; systemctl --user start weston)& disown
```

**Replace with:**
```bash
(sleep 3s; systemctl --user start niri)& disown
```

*Note: You can safely remove the `echo > /home/droid/weston.env` line, as it is no longer needed.*

### Updating `enable_gfxstream`

If you are using Gfxstream for hardware acceleration, you need to update `/usr/local/bin/enable_gfxstream`. Because `niri.service` does not automatically read from `~/weston.env` like `weston.service` did, we must pass the required environment variables directly to the systemd user session before starting Niri.

Open `/usr/local/bin/enable_gfxstream` and modify it as follows:

**1. Remove the `weston.env` creation block:**
```bash
# Remove these lines:
cat << EOF > /home/droid/weston.env

MESA_VK_WSI_DEBUG=sw,linear
XWAYLAND_NO_GLAMOR=1
EOF
```

**2. Update the startup sequence:**
```bash
# The systemd-run command remains the same
sudo systemd-run --collect -E XDG_SESSION_TYPE=wayland --uid=1000 -p PAMName=login -p TTYPath=/dev/tty1 sleep 1d

# Inject the required variables into the systemd user environment
systemctl --user set-environment MESA_VK_WSI_DEBUG=sw,linear XWAYLAND_NO_GLAMOR=1

# Start Niri instead of Weston
(sleep 3s; systemctl --user start niri)& disown

export DISPLAY=:0
export MESA_LOADER_DRIVER_OVERRIDE=zink

# ... keep the rest of the file unchanged
```

## 3. Clean Up Legacy Files (Optional)

You can remove the old Weston environment file to keep your home directory clean:

```bash
rm ~/weston.env
```

## 4. Apply Changes

To apply these changes, either run the modified script directly, or simply close your terminal application and open it again. The `activate_display.sh` profile script will detect the environment, source your updated scripts, and seamlessly launch Niri on the background virtual console.