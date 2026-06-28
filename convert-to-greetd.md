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

# Redirect stdout and stderr for debugging greetd launch
exec 2> "/tmp/weston-wrapper-$(whoami).err"
exec > "/tmp/weston-wrapper-$(whoami).log"

echo "=== weston-wrapper started at $(date) ==="
echo "User: $(whoami) (UID: $(id -u))"
echo "Initial environment:"
env

# Ensure XDG_RUNTIME_DIR is set
if [ -z "$XDG_RUNTIME_DIR" ]; then
    export XDG_RUNTIME_DIR=/run/user/$(id -u)
    echo "Set XDG_RUNTIME_DIR to $XDG_RUNTIME_DIR"
fi

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
echo "Executing weston..."
exec weston --xwayland --shell=desktop-shell.so --continue-without-input
```

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/weston-wrapper
```

> [!IMPORTANT]
> **Wrapper Script Design Notes:**
> * **Permissions-Safe Logging:** Logs are redirected to `/tmp/weston-wrapper-$(whoami).log` using the active username. If the script falls back to running under `_greetd` (for the fallback greeter), it can still write its logs successfully. Redirection to `/home/droid/` would cause the fallback greeter to fail due to permission denied.
> * **XDG_RUNTIME_DIR Fallback:** In virtual/headless environments, the `XDG_RUNTIME_DIR` environment variable might not be immediately available during session startup. The script includes a fallback helper (`export XDG_RUNTIME_DIR=/run/user/$(id -u)`) to ensure Weston initializes correctly.

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

---

## Troubleshooting & Debugging

If the graphical session fails to start or you need to inspect the configuration, use the following resources and techniques:

### 1. Inspect Session Logs
Since stderr and stdout of the wrapper are redirected, you can view real-time logs here:
* **For the autologin session (`droid`):** 
  `cat /tmp/weston-wrapper-droid.log` and `cat /tmp/weston-wrapper-droid.err`
* **For the fallback greeter (`_greetd`):** 
  `cat /tmp/weston-wrapper-_greetd.log` and `cat /tmp/weston-wrapper-_greetd.err`
* **For the `greetd` daemon itself:** 
  `sudo journalctl -u greetd.service -f`

### 2. Force Autologin Re-run (Without Rebooting)
By design, `greetd` tracks whether the `initial_session` has run using a temporary runfile (`/run/greetd.run`). On subsequent restarts of the `greetd` service, it will bypass autologin and fall back to the greeter/fallback session. 

To test changes and force `greetd` to execute the autologin session again, delete the runfile before restarting:
```bash
sudo rm -f /run/greetd.run && sudo systemctl restart greetd.service
```
