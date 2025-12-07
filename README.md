# Linux Troubleshooting – When Curiosity Meets Reality


> **From "What does /proc do?" → To understanding how Linux actually works**

![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Shell Script](https://img.shields.io/badge/Shell_Script-121011?style=for-the-badge&logo=gnu-bash&logoColor=white)

</div>

---

## I Wasn't Trying to Become a Linux Expert


Months ago, I wasn't following any course or tutorial.

I had just installed Ubuntu, and I was curious.

I wanted to see what Linux actually looks like from the inside — not just the desktop, but what's really happening underneath.

So I started exploring the filesystem, step by step:

```
/         → the root, where everything begins
/home     → okay, my files live here
/etc      → lots of configuration files
/var      → logs and temporary stuff
/bin      → commands live here
/sbin     → system commands
/usr      → user programs and libraries
```

Then I reached `/proc`.

It looked... different. Not like other directories. Inside were numbers, strange files, system-looking things. At that time, I didn't fully understand what `/proc` truly was. Later I learned that `/proc` is not a real filesystem stored on disk — it's a **virtual filesystem** that represents running processes and kernel data in real time. It's a window into what the kernel sees.

But at that moment, I only had one thought:

> "Why are there so many numbers here?"

That's when I connected `/proc` with commands like `ps` and `top`. I had learned that `ps` shows running processes and `top` shows them live with CPU and RAM usage. The numbered directories in `/proc` — like `/proc/1234` — are actually live representations of running processes, where `1234` is the PID (Process ID).

I didn't fully understand processes yet, but I knew this:

> "These commands show what's running inside my system right now."

---

## Incident 1: The Day Everything Went Black

### Discovery

One day, out of pure curiosity, I typed:

```bash
top
```

The screen filled with moving numbers:

<div align="center">
<img src="screenshots/incident-01-killed-gnome/01-top-command-original.png" alt="Top Command - First Time Seeing GNOME Processes" width="800"/>

*First time running top - overwhelmed by all the processes, focusing on the one using most resources*
</div>

```
PID   USER  %CPU  %MEM  COMMAND
7515  user   0.3   3.7  gnome-shell
8320  user   0.0   1.0  Xwayland
8350  user   0.0   0.9  gsd-xsettings
```

CPU usage. Memory usage. Process names. PIDs.

At first, it felt overwhelming. But then my eyes locked onto something at the top of the list – a process with `gnome` in its name, `gnome-shell`, using noticeable RAM (3.7%) and a little CPU.

In my beginner brain, the logic was simple and very wrong:

> "Oh... this must be some heavy application eating my memory. Let's kill it and free some RAM."

I didn't think "desktop environment."  
I didn't think "session manager."  
I didn't think "critical system component."

I thought: **"Normal app. Let's kill it."**

So I noted the PID – let's say it was `7515` – and typed:

```bash
kill -9 7515
```

---

### The Crash

The moment I pressed Enter –

**The desktop disappeared.**

No animations.  
No error popup.  
No warning message.

Just... **black.**

For one second I thought maybe the screen just blinked.

But then:
- The mouse cursor didn't move
- Keyboard shortcuts didn't work
- The desktop never came back

I was staring at a **black screen with a blinking cursor** asking for login.

That was the first time in my life I felt real fear from a computer.

My heart started beating fast. I genuinely thought:

> "Okay. I've broken Linux. I finally did it."

There was no browser.  
No YouTube.  
No Stack Overflow.

Only a text screen.

At that time, I didn't even know the word **"TTY"** (TeleTYpewriter – a text-based virtual console). I just knew I was in some raw terminal that felt like the basement of the operating system.

---

### The Struggle

I logged in at that text prompt.

I had no clear plan.  
I didn't know exactly what I had killed.  
I didn't even know what GNOME truly was.

But one thing I remembered: during my filesystem exploration, I had seen directories like `/etc/systemd/` and commands like `systemctl`. I knew that **systemd** manages services.

So my instinct was:

> "If something broke, maybe I can restart it using systemd."

I started trying commands, blindly:

```bash
systemctl status
```

This showed me a tree of all services. I saw things like:

```
● graphical.target - Graphical Interface
     Loaded: loaded
     Active: inactive (dead)
```

So the graphical target was inactive. That made sense – no GUI.

Then I tried:

```bash
systemctl restart gdm
```

**Some error appeared.** I don't remember exactly what, but it didn't work.

I tried:

```bash
systemctl restart graphical.target
```

Still nothing.

Then I did the simplest thing:

```bash
reboot
```

The system rebooted. I waited.

**Still black screen. No GUI.**

My heart sank again. I tried different commands. Different systemd targets. Different services.

<div align="center">
<img src="screenshots/incident-01-killed-gnome/02-systemctl-recovery-attempts.png" alt="Attempting systemctl Recovery Commands" width="800"/>

*Desperately trying systemctl commands to bring back the GUI - checking gdm3 status*
</div>

```bash
systemctl restart gdm3
systemctl isolate graphical.target
```

Rebooted again.  
Still nothing.

**I was in pure survival mode** – trial and error, fear and curiosity mixed together. At no point was I following a tutorial. I was just trying to survive.

Then – after one of the reboots, after trying multiple different systemd commands – **the desktop finally came back.**

Login screen appeared.  
I logged in carefully, expecting something to be broken.

But... everything was normal. The desktop was back. All my files were there. Nothing was lost.

That moment felt unreal. I didn't feel like I had "learned Linux."

**I felt like I had escaped from a cave.**

---

### What I Learned (Then)

That day I realized something deep, even if I couldn't explain it in technical words yet:

> "Sometimes breaking things teaches you more in 10 minutes than reading for 10 days."

**What I understand now:**

| Component | What It Does | What Happens When Killed |
|-----------|--------------|--------------------------|
| `gnome-shell` | Compositor and window manager – renders everything you see | Entire graphical layer disappears |
| SIGKILL (`-9`) | Most forceful termination signal | Process dies immediately, no cleanup |
| TTY | Text-based virtual console | Always running, independent of GUI |
| systemd | Init system (PID 1) managing all services | Can restart crashed services |
| Boot sequence | Multiple reboots restored services in correct order | Fresh start fixes broken states |

---

## Incident 2: The Boot That Never Came

### When Everything Changed

Then suddenly, during one of my reboots while experimenting with killing GNOME processes, something completely different happened.

Instead of the login screen, I saw **text scrolling on a black screen**:

<div align="center">
<img src="screenshots/incident-02-boot-failure/01-timeout-errors.png" alt="Boot Timeout Errors" width="800"/>

*Boot sequence hanging - systemd waiting for non-existent disk UUIDs*
</div>

```
[  OK  ] Started User Manager for UID 1000.
[  OK  ] Reached target Multi-User System.
[ TIME ] Timed out waiting for device /dev/disk/by-uuid/7DA29C4A4455453E.
[DEPEND] Dependency failed for /mnt/sda2.
[ TIME ] Timed out waiting for device /dev/disk/by-uuid/8882B4B2B844000C.
[DEPEND] Dependency failed for /mnt/sda1.
[  ***] A start job is running for /mnt/sda2 (47s / no limit)
```

Then the system just froze with a blinking underscore:

```
_
```

Now I knew instantly: **This wasn't GNOME. This wasn't a session problem. This was boot.**

The system wasn't reaching the desktop at all. This was happening **before** the GUI even had a chance to start. The boot sequence was stuck waiting for something.

I had accidentally stumbled into a completely different problem while experimenting with GUI crashes.

---

### The Diagnosis

The error messages were clear:

```
Timed out waiting for device /dev/disk/by-uuid/...
  ↓
systemd was trying to mount a disk
  ↓
Dependency failed for /mnt/sda2
  ↓
The mount operation failed
  ↓
The UUID didn't exist
```

After waiting and seeing no progress, the system eventually dropped me into **emergency mode**:

<div align="center">
<img src="screenshots/incident-02-boot-failure/02-tty-emergency-login.png" alt="TTY Login in Emergency Mode" width="800"/>

*Dropped into TTY3 after boot failure - emergency mode access*
</div>

```
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or "exit"
to boot into default mode.
Give root password for maintenance
(or press Control-D to continue):
```

I entered the root password.

I knew exactly where to look. Boot-related mount problems live in one place:

```bash
nano /etc/fstab
```

---

### The Problem

When I opened it, I saw this at the bottom:

<div align="center">
<img src="screenshots/incident-02-boot-failure/03-broken-fstab.png" alt="Broken fstab Configuration" width="800"/>

*The problematic /etc/fstab with old external HDD mount entries*
</div>

```
# /etc/fstab: static file system information.
UUID=abc123...  /                    ext4  errors=remount-ro  0  1
UUID=def456...  /boot/efi            vfat  umask=0077         0  1
UUID=8B82B4B2B844000C  /mnt/sda1    ntfs  defaults,auto,users,rw,nofail  0  0
UUID=7DA29C4A4455453E  /mnt/sda2    ntfs  defaults,auto,users,rw,nofail  0  0
```

Those last two lines – old mount entries for my **1TB external HDD** that was no longer connected. The UUIDs were from partitions that either got reformatted or were on a drive that's gone.

**What fstab does:**

| Column | Purpose | Example |
|--------|---------|---------|
| UUID | Permanent disk identifier | `UUID=7DA29C4A...` |
| Mount Point | Where to mount | `/mnt/sda2` |
| Filesystem Type | ext4, ntfs, vfat, etc. | `ntfs` |
| Options | Mount behavior | `defaults,nofail` |
| Dump | Backup flag (0 or 1) | `0` |
| Pass | fsck order (0, 1, or 2) | `0` |

So at boot, systemd tried to mount those UUIDs. When it couldn't find them, it waited (hence "Timed out...") and eventually gave up, dropping me into emergency mode.

---

### The Fix

I deleted those last two lines.

<div align="center">
<img src="screenshots/incident-02-boot-failure/04-fixed-fstab.png" alt="Fixed fstab Configuration" width="800"/>

*Fixed /etc/fstab with problematic UUID entries commented out*
</div>

Then I did something important – I **tested** the fstab without rebooting:

```bash
mount -a
```

**Critical command:** `mount -a` tries to mount everything in fstab. If there's an error, you'll see it immediately without needing to reboot and potentially break boot again.

No errors appeared. Good.

Then:

```bash
reboot
```

The system rebooted. No timeout messages. No dependency failures.

The desktop came back normally.

---

### What I Understood

At that exact moment, I understood something very clearly:

> "Earlier, I thought I was playing with the GUI. But here, I had touched the boot chain itself."

This was deeper than GNOME. This was **systemd's mount system** – the part of the boot sequence that happens before the GUI even exists.

**Boot process stages:**

```
1. BIOS/UEFI loads bootloader (GRUB)
   ↓
2. GRUB loads kernel
   ↓
3. Kernel starts and mounts root filesystem
   ↓
4. Kernel starts systemd (PID 1)
   ↓
5. systemd reads fstab and mounts all filesystems  ← I BROKE THIS
   ↓
6. systemd starts services (network, display manager, etc.)
   ↓
7. Display manager (GDM) starts
   ↓
8. You see login screen
```

I had interrupted step 5, so steps 6-8 never happened.

---

## Incident 3: The Desktop That Wouldn't Die

### The Return

Weeks passed.

Today, that memory came back to me.

I was again inside Ubuntu – but this time **Ubuntu 24.04** (back then it was 22.04). Again playing with the terminal. And suddenly I thought:

> "Let me try that again. Let me crash the GUI again – but this time intentionally."

This time, I was more conscious. I knew about `top`. I knew about `ps`. I knew about PIDs. I wasn't just blindly typing anymore. I understood that processes have states, signals, and relationships. I knew that `kill -9` is SIGKILL – the nuclear option.

---

### The Experiment Begins

So I opened:

```bash
top
```

<div align="center">
<img src="screenshots/incident-03-auto-recovery/01-top-gnome-processes.png" alt="Top Command Showing GNOME Processes" width="800"/>

*Live system monitoring - gnome-shell at the top consuming resources*
</div>

Again, I saw all the running processes live. CPU, RAM, everything. And again, near the top, I saw processes related to GNOME taking the most resources:

```
PID   USER  %CPU  %MEM  COMMAND
12520 user   0.3   3.3  gnome-shell
13011 user   0.3   0.1  gvfs-afc-volume
13162 user   0.3   0.6  gjs
```

This time I knew the names:

- **gnome-shell** - The compositor and UI shell
- **mutter** - The window manager backend
- **gnome-session-binary** - The session manager
- **Xwayland** - The compatibility layer for X11 apps

I picked `gnome-shell`. I copied the PID – `12520`.

<div align="center">
<img src="screenshots/incident-03-auto-recovery/02-kill-gnome-shell.png" alt="Killing gnome-shell Process" width="800"/>

*Executing kill -9 12520 to terminate gnome-shell*
</div>

I typed:

```bash
kill -9 12520
```

The screen **blinked** for a second.

Then... **the login screen appeared.**

<div align="center">
<img src="screenshots/incident-03-auto-recovery/03-login-screen-recovery.png" alt="GDM Login Screen After Kill" width="800"/>

*GNOME Display Manager automatically restarted - controlled recovery instead of crash*
</div>

Not a black dead screen.  
Not a frozen system.

Just the normal login screen, as if I had logged out intentionally.

I logged in again.  
Desktop came back. Everything normal.

I felt confused.

> "Wait... so this time it didn't really crash?"

---

### Systematic Testing

I thought maybe I picked the wrong process, or maybe I needed to kill multiple things. So I went back and tried systematically:

**Attempt 1: Killing GDM3**

<div align="center">
<img src="screenshots/incident-03-auto-recovery/04-kill-gdm3-attempt.png" alt="Attempting to Kill GDM3" width="800"/>

*Trying to kill the display manager - required sudo privileges*
</div>

```bash
kill -9 1715
bash: kill: (1715) - Operation not permitted
sudo kill -9 1715
```

Result: **1-second freeze → back to login screen.**

---

**Attempt 2: Killing gnome-session-binary**

<div align="center">
<img src="screenshots/incident-03-auto-recovery/05-kill-gnome-session.png" alt="Killing gnome-session-binary" width="800"/>

*Multiple GNOME session processes shown - attempting to kill PID 7487*
</div>

```bash
kill -9 7487
```

Result: **Brief flicker → login screen → normal system.**

---

**Attempt 3: Killing Xwayland**

<div align="center">
<img src="screenshots/incident-03-auto-recovery/06-kill-xwayland.png" alt="Killing Xwayland Process" width="800"/>

*Terminating Xwayland (PID 8320) - X11 compatibility layer*
</div>

```bash
kill -9 8320
```

Result: **Some applications flickered → desktop stayed up.**

---

**Attempt 4: Killing mutter**

<div align="center">
<img src="screenshots/incident-03-auto-recovery/07-kill-mutter.png" alt="Killing mutter Window Manager" width="800"/>

*Killing the window manager backend (PID 8410)*
</div>

```bash
kill -9 8410
```

Result: **Brief freeze → auto-recovery.**

---

**Additional Attempts:**

<div align="center">
<img src="screenshots/incident-03-auto-recovery/08-kill-various-processes.png" alt="Additional Kill Attempts" width="800"/>

*Systematic testing - killing gnome-shell PID 5413*
</div>

<div align="center">
<img src="screenshots/incident-03-auto-recovery/09-tty-kill-attempt.png" alt="TTY-based Kill Attempt" width="800"/>

*Another attempt from TTY - PID 3073*
</div>

**Every single time, the same pattern:**

```
Brief flicker → login screen → normal system
```

The desktop kept coming back. It refused to stay dead.

No matter what I killed, Ubuntu just refused to fully die.

---

### The Discovery

That's when it finally clicked.

Months ago, when I first crashed the GUI, I was likely on **Ubuntu 22.04** (or possibly even 20.04).

Today, I'm on **Ubuntu 24.04**.

**The system has fundamentally changed between these versions.**

I checked the GDM service configuration:

```bash
systemctl status gdm3
```

Output showed:

```
● gdm.service - GNOME Display Manager
     Loaded: loaded (/usr/lib/systemd/system/gdm.service; static)
     Active: active (running) since Wed 2025-12-03 11:33:41 IST; 6min ago
```

And when I checked the service file itself:

```bash
systemctl cat gdm3
```

I would see (typical configuration):

```
[Unit]
Description=GNOME Display Manager
...

[Service]
Type=notify
ExecStart=/usr/sbin/gdm3
Restart=always
RestartSec=1s
...
```

That `Restart=always` line is the key. It tells systemd: **"If this service dies for any reason, restart it automatically after 1 second."**

---

## What These Incidents Taught Me

### The Complete Journey

Looking back at this entire journey:

1. **Months ago**: Accidentally killed GNOME → black screen panic → struggled with recovery → multiple reboots → finally restored
2. **Today**: Tried to recreate that crash → everything auto-recovered → couldn't break it
3. **Today** (during experiments): Unexpected boot failure from old fstab entries → emergency mode → quick fix
4. **Today** (after boot fix): Continued GUI experiments → confirmed modern auto-healing

At no point was I following a tutorial.  
At no point was I trying to build a "project."

I was just asking one question again and again:

> "What happens if I do this?"

And Linux answered every time – sometimes gently with a login screen, sometimes brutally with a black screen and emergency mode.

---

## Technical Skills Demonstrated

### Process Management
- Understanding PIDs and process hierarchy
- Process states and signals (SIGTERM vs SIGKILL)
- Using `ps`, `top`, `kill`, `killall`
- The `/proc` filesystem as a window into running processes

### systemd Service Management
- Service control (`systemctl start/stop/restart/status`)
- Service configuration files and restart policies
- Understanding targets (graphical.target)
- User services vs system services
- Auto-restart mechanisms (`Restart=always`)

### Boot Process & Recovery
- GRUB → Kernel → systemd boot sequence
- Boot stages and dependencies
- Emergency mode access and usage
- Reading boot logs with `journalctl -xb`
- Understanding why boot can fail

### Filesystem & Mounts
- `/etc/fstab` syntax, purpose, and critical role in boot
- UUID-based disk identification vs device names
- Using `mount -a` to test configurations safely
- Mount dependencies in systemd boot process
- How disconnected drives can break boot

### Desktop Environment Architecture
- Display managers (GDM) and their role
- Compositors and window managers (gnome-shell, mutter)
- Session managers (gnome-session-binary)
- Wayland vs X11 differences
- Process supervision and auto-recovery

### Recovery Techniques
- TTY navigation (Ctrl+Alt+F1-F6)
- Emergency mode troubleshooting workflow
- Service restoration methods
- System recovery without data loss
- Testing changes before committing (mount -a before reboot)

---

### Troubleshooting Methodology I Developed

```
1. Stay calm
   ↓
   Panic wastes time and clouds judgment

2. Identify the layer
   ↓
   Is this GUI, boot, network, or something else?

3. Check logs
   ↓
   journalctl, dmesg, service status give clues

4. Test safely
   ↓
   Validate changes before committing (like mount -a)

5. Document
   ↓
   Record what worked for next time

6. Understand, don't memorize
   ↓
   Know WHY it broke, not just HOW to fix
```

---

## Commands I Practice

### Process Monitoring and Control
```bash
# Real-time process viewer with CPU/memory
top

# Find specific processes
ps aux | grep <process>

# Force kill process (SIGKILL - nuclear option)
kill -9 <PID>

# Kill all instances by name
killall <process-name>

# Find PID by process name
pgrep <process-name>
```

### systemd Service Management
```bash
# Check service status and recent logs
systemctl status <service>

# Restart a service
systemctl restart <service>

# Stop a service
systemctl stop <service>

# Start a service
systemctl start <service>

# View service configuration file
systemctl cat <service>

# Check restart policy
systemctl show <service> -p Restart

# Switch to graphical mode
systemctl isolate graphical.target

# List user services
systemctl --user list-units

# Show active login sessions
loginctl list-sessions
```

### Boot and Recovery
```bash
# View boot logs with explanations
journalctl -xb

# Test fstab without reboot (CRITICAL)
mount -a

# Edit filesystem table
nano /etc/fstab

# Show disk UUIDs and filesystem types
blkid

# Exit emergency mode to normal boot
systemctl default
```

### System Information
```bash
# Check if using Wayland or X11
echo $XDG_SESSION_TYPE

# Show current TTY number
tty

# Show kernel version
uname -r

# Show Ubuntu version
lsb_release -a

# Show GNOME Shell version
gnome-shell --version
```

---

## Connect With Me

I'm actively learning and seeking opportunities in Linux administration and cloud engineering.

[![Email](https://img.shields.io/badge/Email-jillahir9999%40gmail.com-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:jillahir9999@gmail.com)
[![GitHub](https://img.shields.io/badge/GitHub-jillravaliya-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/jillravaliya)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Jill_Ravaliya-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/jill-ravaliya-684a98264)

</div>

### Looking For

- Junior Linux Administrator roles
- Entry-level Cloud Engineer positions (trainee/intern level)
- Linux Support Engineer roles
- Infrastructure internships
- DevOps trainee positions

---

> **"Sometimes breaking things teaches you more in 10 minutes than reading for 10 days."**

---

**Final Note**: This documents real desktop Linux troubleshooting on my personal Ubuntu system. This is not production cloud experience yet – I'm building foundational skills to transition into cloud engineering roles.

</div>
