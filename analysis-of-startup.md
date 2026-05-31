sudo systemd-run --collect -E XDG_SESSION_TYPE=wayland --uid=1000 -p PAMName=login -p TTYPath=/dev/tty1 sleep 1d

When you execute that specific systemd-run command, you are essentially bypassing the traditional, heavy Linux display manager (like GDM or SDDM) and manually building a secure, authenticated graphical user session from scratch.

Here is the exact step-by-step flow of how systemd and the low-level Linux plumbing layers interact, process your arguments, and ultimately get "tricked" into treating a background sleep command as a legitimate local, interactive graphical login session.

Step 1: The Blueprint and Handshake (systemd-run)
When you hit enter, systemd-run talks to the systemd system manager (PID 1) over D-Bus.

The Action: It requests the creation of a temporary, dynamic service unit (e.g., run-r123456789.service).

The Plumbing Interaction: Because you passed --collect, systemd flags this unit in its memory space as ephemeral. It tells the manager: "Do not leave a failed or dead unit history behind when this process exits. Wipe it completely from the cgroup tree the second it's done."

Step 2: Privilege Allocation and Sandbox Routing (--uid=1000)
Before systemd executes any binaries, it sets up the security context for the new process.

The Action: systemd allocates the execution to User ID 1000.

The Plumbing Interaction: Crucially, because you are using PAMName=, systemd does not immediately drop privileges to UID 1000 yet. PID 1 stays as root for a brief moment longer because initializing a PAM session requires root permissions to read security limits and write system logs.

Step 3: Directing the Virtual Terminal (-p TTYPath=/dev/tty1)
Your service needs a physical or virtual space to attach its standard input, output, and error streams.

The Action: Systemd opens /dev/tty1 and duplicates it across the file descriptors (0, 1, and 2) of the service.

How the Plumbing is Tricked: Normally, background daemons have their streams routed to /dev/null or the journal. By binding the service to a physical virtual console (/dev/tty1), you are making the system think there is a real screen and keyboard context attached to this process. This satisfies low-level graphics subsystems (like KMS/DRM kernel drivers) that require an associated TTY controlling terminal to grant direct access to video hardware.

Step 4: Activating the Authentication Stack (-p PAMName=login)
This is where the actual "trick" occurs. Systemd invokes the Pluggable Authentication Module architecture.

[systemd-run] ──> [systemd (PID 1)] ──> Opens PAM Stack (login)
                                                │
                                                ▼
                                        [pam_systemd.so]
                                                │
                                                ▼
                                    Triggers [systemd-logind]
                                 (Registers active Seat0 Session)
The Action: systemd calls pam_start("login", "your_user_name", ...) and steps through /etc/pam.d/login.

The Plumbing Interaction: 1. pam_securetty.so & pam_env.so: The system checks security policies for tty1. It loads standard system environment variables.
2. pam_limits.so: The kernel sets up the process's resource limits (max open files, memory limits) based on user-space configuration files rather than daemon defaults.
3. pam_systemd.so: This module intercepts the request and signals systemd-logind (the seat and session manager daemon).

How systemd-logind is Tricked: logind looks at the incoming PAM request, sees that it originated from a login service stack, and sees it is attached to /dev/tty1. It says: "Ah, a user just physically logged into TTY1!" * The Result: logind assigns a unique Session ID, attaches it to seat0 (your physical GPU, keyboard, and mouse), and automatically mounts and provisions /run/user/1000 (the $XDG_RUNTIME_DIR).

Step 5: Injecting the Graphical Context (-E XDG_SESSION_TYPE=wayland)
Now that PAM has set up the core user environment, systemd injects your explicit environment modification right before spawning the final application.

The Action: Systemd appends XDG_SESSION_TYPE=wayland into the environment block.

How the Plumbing is Tricked: Under a standard login, pam_systemd.so or your desktop display manager determines what kind of session you have and sets this variable. By forcing it manually here, you are overriding system detection.

The Impact: Any child process spawned under this unit that queries the environment will instantly believe it is running inside a modern Wayland display environment. If you replace sleep 1d with a Wayland compositor (like sway, hyprland, or weston), it will skip X11 fallback checks entirely and immediately attempt to bind to the Wayland protocols.

Step 6: Executing the Payload (sleep 1d)
Finally, systemd performs an execve() system call to transition from the systemd setup routine into your actual payload.

The Action: The process formally drops its root privileges, switches completely to UID 1000, and executes sleep 1d.

The Current State: You now have a process running as an ordinary user, holding open a perfectly valid, fully authenticated logind seat session on TTY1, complete with access to local hardware and runtime directories. It will sit there perfectly idle for 24 hours, keeping that entire fake login environment alive and open for any other operations you want to multiplex into that session.
