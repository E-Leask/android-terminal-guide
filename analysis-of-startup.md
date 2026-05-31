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


This step represents the handoff where your environment preparation ends and the actual graphical display pipeline begins. The two commands work together to launch the display server cleanly in a headless/virtual container environment and bridge it back to your interactive terminal session (/dev/pts/1).

1. Breaking Down: (sleep 3s; systemctl --user start weston)& disown
This complex command executes a background handoff to the system's initialization manager while protecting the process from terminal hangups.

       ( sleep 3s  ;  systemctl --user start weston ) & disown
         ────┬───     ───────────────┬──────────────  ┬   ──┬──
             │                       │                │     │
  1. Race-condition protection       │                │  4. Shell decoupling
                                     │                │
                        2. Session-isolated launch    │
                                                      │
                                           3. Background routing
The Sequential Group (...): The parentheses group multiple commands together so they execute sequentially in order as a single unit.

The 3-Second Pause (sleep 3s): This is a critical race-condition protection. It forces a tiny pause to ensure that the heavy systemd-run command executed right before it has completely finished initializing the PAM session, talking to the kernel, and generating the /run/user/1000 directory. If Weston tries to start at millisecond zero before that folder exists, it will crash instantly.

The User-Level Service (systemctl --user start weston): This instructs systemd to launch Weston. Because the --user flag is supplied, it bypasses the system administrator space and talks directly to the systemd instance owned by the droid user account. This ensures that the display server runs inside the exact privilege boundary as the user, satisfying Linux's strict security rule that the owner of the graphical window server must match the owner of the applications trying to write to it.

The Background Ampersand (&): This immediately pushes the entire grouped execution blocks into the background, returning control of the terminal prompt back to you so the script can finish executing without hanging up.

The Process Decoupling (disown): Normally, background tasks remain children of the active terminal shell. If you close your terminal window app, the shell pushes a hangup signal (SIGHUP) down the process tree, killing its children. By calling disown, you remove Weston from the shell's tracking list.

🛡️ The Interconnected Invisible Net
While disown prevents your shell from killing Weston when the window closes, the persistent systemd-run session we discussed earlier on /dev/tty1 acts as the operating system's safety net. Even when your active terminal window exits, systemd-logind looks at the system, sees that the user droid still has a running login process on tty1, and refuses to sweep away the critical graphical socket directories (/run/user/1000).

2. Breaking Down: export DISPLAY=:0
This command bridges the isolation gap between two completely separate terminal environments within your system.

The Isolation: When you access your Android terminal app, you are typing into a software-generated pseudo-terminal slave interface (/dev/pts/1). Meanwhile, your authenticated graphics sandbox is living over on a virtual system console space (/dev/tty1). Your shell on pts/1 knows absolutely nothing about the graphics loop running on tty1.

The Identifier (:0): In the Linux X11/XWayland compatibility architecture, :0 stands for "the very first local graphical display screen on this machine."

The Bridge (export): By defining and exporting DISPLAY=:0, you place a persistent signpost inside your current interactive shell environment.

When you later try to launch a GUI application (like a browser or an IDE) from your command line on /dev/pts/1, that application checks its environment block, reads DISPLAY=:0, and says: "Ah, I shouldn't throw an error or print text to this command line; I need to route my graphical output instructions to the display server running on local screen zero."
