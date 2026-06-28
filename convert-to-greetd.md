# Converting Weston to Launch via Greetd (Architectural Limitations & Recovery)

> [!CAUTION]
> **UNSTABLE CONFIGURATION WARNING:**
> Configuring `greetd` to launch Weston automatically at boot inside this virtualized Android terminal environment is **inherently unstable** and can lead to a **complete crash of the host terminal application** and guest VM panic/reboot.
>
> It is strongly recommended to use the original on-demand display initialization. This document outlines why this limitation exists and how to completely revert the system to the stable original configuration.

---

## Why Greetd Autostart Fails (Architectural Analysis)

In a standard Linux desktop environment, the GPU and DRM subsystems are initialized by the kernel at boot, allowing `greetd` to safely claim `tty1` and run the compositor. 

However, the Android terminal virtualized environment operates under a guest-host model:
1. **Host Display Activity Dependency:** The guest VM’s `virtio-gpu` driver depends on the host Android Display Activity being actively open and connected.
2. **Early Boot Race Conditions:** When `greetd.service` starts automatically during early boot, it attempts to claim the DRM seat (`/dev/dri/card0`) and perform atomic page commits while the host display activity is inactive.
3. **VM Panics & Host Crashes:** This leads to guest kernel-level DRM atomic commit failures (`atomic: couldn't commit new state: Permission denied`), resulting in a guest VM hang/panic, or completely crashing the host Android terminal application when a user attempts to open the display tab.
4. **On-Demand Design:** The original display scripts avoided this by delaying Weston's start until the user logs into an interactive shell (triggering `/etc/profile.d/activate_display.sh`). This guarantees that the host display channel is open, powered, and ready to receive frames.

---

## Steps to Revert to the Stable Configuration

If you have applied the `greetd` configuration and are experiencing system restarts or display crashes, follow these steps to restore the original stable setup.

### 1. Stop and Disable the Greetd Service

Stop the running `greetd` instance and disable it from launching at boot:

```bash
# Stop the daemon immediately
sudo systemctl stop greetd.service

# Disable greetd from boot execution
sudo systemctl disable greetd.service
```

### 2. Restore User systemd Manager and Services

Make sure the user-level systemd manager is running and unmask the default user-space Weston units:

```bash
# Ensure the user manager is active
sudo systemctl start user@1000.service

# Unmask the systemd user Weston service and socket
XDG_RUNTIME_DIR=/run/user/1000 systemctl --user unmask weston.service weston.socket
```

### 3. Restore the Original Shell Profile Script

Remove the custom greetd profile and restore the original on-demand display activation script from the backup directory:

```bash
# Restore the profile script
sudo cp /home/droid/android-terminal-guide/configs/activate_display.sh /etc/profile.d/activate_display.sh

# Ensure execution permissions
sudo chmod +x /etc/profile.d/activate_display.sh
```

### 4. Remove the Wrapper Script & Greetd Backup Files

Clean up the temporary wrapper script and greetd backup folders:

```bash
# Remove the wrapper binary
sudo rm -f /usr/local/bin/weston-wrapper

# Remove the temporary greetd configuration backups
rm -rf /home/droid/android-terminal-guide/configs/greetd
```

### 5. Uninstall Greetd Package (Optional)

To fully clean up greetd and its unprivileged system user (`_greetd`) from the system:

```bash
sudo apt purge greetd -y
sudo apt autoremove -y
```

---

## Verifying Restoration & Starting Display

### 1. Interactive Shell Startup
Once reverted, open a new interactive terminal window or source the profile script to trigger the display startup:

```bash
source /etc/profile.d/activate_display.sh
```

You should see the message:
`Display is enabled. Please open a display activity before running any GUI applications.`

### 2. Manual Compositor Start (For Existing/Active Shells)
If you are in an existing shell session where you just completed the reversion and want to start Weston right now without logging out or closing the terminal:

```bash
# Source the display profile
source /etc/profile.d/activate_display.sh

# Force-start the user-space Weston compositor under systemd user manager
XDG_RUNTIME_DIR=/run/user/1000 systemctl --user start weston.service
```

### 3. Open the Display
Navigate to your terminal's **Display Activity** tab, and then launch your GUI application (e.g. `chromium`). It will render cleanly on screen without crashing the application.

