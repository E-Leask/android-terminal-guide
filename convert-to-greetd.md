# Converting Weston to Launch via Greetd

This guide outlines the steps to configure the system to launch Weston automatically at boot using the `greetd` login/session manager, replacing the default `systemd-run` and profile-based auto-launch mechanism.

## Overview

By default, the Android terminal environment initializes its graphical session on `tty1` via a combination of a `systemd-run sleep 1d` session (triggered from `/etc/profile.d/activate_display.sh`) and user-level systemd services (`weston.service`/`weston.socket`).

Migrating to `greetd` provides a cleaner, standard Linux session architecture:
1. **Native Session Management:** `greetd` manages PAM login sessions and the lifecycle of the graphical server natively.
2. **True System Service:** Weston starts automatically at boot via `greetd.service` rather than waiting for an interactive user shell login.
3. **No Background systemd-run Hacks:** Bypasses the need for spawning dummy `sleep` commands on virtual terminals.

---

## 1. Stop and Disable Legacy Weston Services

First, stop the existing Weston user-level services and disable the automatic profile script that initiates them.

```bash
# Disable the interactive login autostart script
sudo rm /etc/profile.d/activate_display.sh

# Stop and mask the systemd user Weston service and socket
systemctl --user stop weston.service weston.socket
systemctl --user mask weston.service weston.socket
```

---

## 2. Install Greetd

Install the `greetd` daemon from the Debian package manager:

```bash
sudo apt update
sudo apt install greetd -y
```

---

## 3. Create the Weston Wrapper Script

Because the graphics environment configuration depends on whether hardware acceleration via Gfxstream is enabled, we create a wrapper script to export the proper environment variables before executing Weston.

Create `/usr/local/bin/weston-wrapper`:

```bash
sudo nano /usr/local/bin/weston-wrapper
```

Add the following contents:

```bash
#!/bin/bash

# Detect if Gfxstream is enabled via kernel cmdline and export appropriate graphics variables
if grep -q -w "gfxstream_enabled" /proc/cmdline; then
    export MESA_LOADER_DRIVER_OVERRIDE=zink
    export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json
    export MESA_VK_WSI_DEBUG=sw,linear
    export XWAYLAND_NO_GLAMOR=1
    export LIBGL_KOPPER_DRI2=1
else
    export MESA_LOADER_DRIVER_OVERRIDE=zink
    export LIBGL_ALWAYS_SOFTWARE=1
    export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.json
fi

# Launch Weston with support for XWayland and headless environments
exec weston --xwayland --shell=desktop-shell.so --continue-without-input
```

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/weston-wrapper
```

---

## 4. Configure Greetd

Greetd configuration is managed via `/etc/greetd/config.toml`. Configure `greetd` to auto-login the `droid` user and run the Weston wrapper script on boot.

Open the configuration file:

```bash
sudo nano /etc/greetd/config.toml
```

Replace or update its content to match:

```toml
[terminal]
# The Virtual Terminal (VT) to run on
vt = 1

[default_session]
# The fallback session (greeter) to run if the user logs out or if initial_session fails
command = "agreety --cmd /usr/local/bin/weston-wrapper"
user = "_greetd"

[initial_session]
# Automatically log in the 'droid' user to Weston on system boot
command = "/usr/local/bin/weston-wrapper"
user = "droid"
```

> [!NOTE]
> Debian systems typically use `_greetd` as the unprivileged system user for the login manager. If your system uses a different user (e.g. `greeter` or `greetd`), update the `user` field in `[default_session]` accordingly.

Ensure the `droid` user has correct groups to access the display and input devices (this is usually configured by default):
```bash
sudo usermod -aG video,renderer,input droid
```

---

## 5. Configure the Shell Environment

Since Weston is now started by `greetd` as a background PAM session on `tty1`, any new interactive terminal opened by the user (which runs on a pseudo-terminal `/dev/pts/*`) must know how to locate and connect to the Wayland/X11 display.

Create or update `/etc/profile.d/activate_display.sh` to configure these environment variables dynamically:

```bash
sudo nano /etc/profile.d/activate_display.sh
```

Add the following contents:

```bash
#!/bin/bash

# Only configure for the 'droid' user in interactive shell sessions
if [[ "$USER" == "droid" && -n "$PS1" ]]; then
  # Point applications to the Wayland display and XWayland server created by greetd's Weston
  export WAYLAND_DISPLAY=wayland-0
  export DISPLAY=:0

  # Setup the graphics driver environment matching the system hardware
  if grep -q -w "gfxstream_enabled" /proc/cmdline; then
    export MESA_LOADER_DRIVER_OVERRIDE=zink
    export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json
    export MESA_VK_WSI_DEBUG=sw,linear
    export XWAYLAND_NO_GLAMOR=1
    export LIBGL_KOPPER_DRI2=1
  else
    export MESA_LOADER_DRIVER_OVERRIDE=zink
    export LIBGL_ALWAYS_SOFTWARE=1
    export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.json
  fi
  
  echo "Display environment variables configured for greetd-managed Weston session."
fi
```

Make the profile script executable:

```bash
sudo chmod +x /etc/profile.d/activate_display.sh
```

---

## 6. Apply Changes

Enable and start the `greetd` systemd service:

```bash
# Enable greetd to run at system boot
sudo systemctl enable greetd.service

# Start greetd immediately
sudo systemctl start greetd.service
```

If you are already in an active login session, you can restart your terminal or source the profile script to access the display:

```bash
source /etc/profile.d/activate_display.sh
```
