# 🐧 Linux for DevOps Engineers — Interview-Ready Study Notes
## Sections 4–6: Process Management, Package Management & Networking

---

# 🔴 SECTION 4 — PROCESS MANAGEMENT

---

## 14. Processes & Signals

### 📖 Concept Explanation

A **process** is a running instance of a program. Every process has a unique **PID (Process ID)**. The kernel tracks everything about a process: its state, open files, memory mappings, CPU time, and parent/child relationships.

**Process Lifecycle** — how processes are created and die:
- **fork()**: Creates an exact copy of the parent process (child inherits file descriptors, memory). Almost all process creation goes through fork.
- **exec()**: Replaces the current process image with a new program. fork() + exec() = "run a new program."
- **wait()**: Parent waits for child to finish and collects its exit code.
- **exit()**: Process terminates, releasing resources.

**Process Identity**:
- **PID**: Process ID — unique number assigned by kernel
- **PPID**: Parent Process ID — who spawned this process
- **UID/GID**: User/Group that owns the process (determines permissions)
- **EUID**: Effective UID (differs from UID when SUID is in play)

**Process States**:
| State | Symbol | Meaning |
|---|---|---|
| Running | R | Actively using CPU or in run queue |
| Sleeping (interruptible) | S | Waiting for event (I/O, timer, signal) — can be interrupted by signal |
| Sleeping (uninterruptible) | **D** | Waiting on I/O — **cannot** be interrupted, not even by SIGKILL |
| Stopped | T | Paused by SIGSTOP or debugger |
| Zombie | **Z** | Finished but parent hasn't called wait() — entry remains in process table |
| Dead | X | Being removed (transient, rarely seen) |

> **Critical**: D-state processes are the most dangerous in production — they cannot be killed. They indicate a blocking I/O wait (NFS hang, failing disk, storage backend issue). The only fix is resolving the I/O problem or rebooting.

### 🎯 Why It Is Used

Process management is core to running stable services. Understanding signals is critical for:
- **Zero-downtime reloads**: Send SIGHUP to nginx/HAProxy to reload config without dropping connections
- **Graceful shutdowns**: systemd sends SIGTERM (graceful) then SIGKILL (force) with a timeout
- **Debugging**: Pausing a process with SIGSTOP to inspect its state
- **CI/CD**: Scripts that need to handle signals and clean up before terminating

### 🏗️ Architecture Diagram

```
PROCESS LIFECYCLE:
                    fork()
  ┌────────────┐ ──────────▶ ┌────────────┐
  │   PARENT   │             │   CHILD    │ ──→ exec() → runs new program
  │  (bash)    │             │  (copy)    │
  └────────────┘ ◀────────── └────────────┘
                    wait()          │
                  (collects          │ exit()
                  exit code)         ▼
                              [process terminates]
                              Parent collects exit code
                              → If parent doesn't wait():
                                child becomes ZOMBIE (Z)

PROCESS TREE (pstree):
  systemd (PID 1)
  ├── sshd
  │   └── sshd (session)
  │       └── bash
  │           └── top
  ├── nginx (master)
  │   ├── nginx (worker)
  │   └── nginx (worker)
  └── cron
      └── sh (running job)

SIGNAL FLOW:
  kill -SIGTERM <pid>
       │
       ▼
  ┌────────────────────────────────────────────────┐
  │  Process Signal Handler (if registered)         │
  │  → Custom cleanup code runs                     │
  │  → Process exits gracefully                     │
  └────────────────────────────────────────────────┘
       │
  If handler not registered or SIGKILL:
       ▼
  ┌────────────────────────────────────────────────┐
  │  Kernel default action (terminate/stop/ignore)  │
  └────────────────────────────────────────────────┘
```

### ⚙️ Key Commands

```bash
# ─── Signals Reference ────────────────────────────────────────
# SIGTERM  (15) — graceful termination request (catchable, process can clean up)
# SIGKILL   (9) — force kill (uncatchable, kernel kills immediately — no cleanup)
# SIGHUP    (1) — hangup (traditionally: reload config; disconnect terminal)
# SIGINT    (2) — interrupt (Ctrl+C — same as SIGTERM but from keyboard)
# SIGSTOP  (19) — pause process (uncatchable)
# SIGCONT  (18) — continue paused process
# SIGUSR1/2 (10/12) — user-defined signals (app-specific, e.g., rotate logs)
# SIGCHLD  (17) — child process changed state (parent receives this)

kill -l                     # list all signal names and numbers
kill -SIGTERM 1234          # graceful terminate PID 1234
kill -15 1234               # same (by number)
kill -9 1234                # force kill (use only when SIGTERM fails)
kill -SIGHUP $(pgrep nginx) # reload nginx config
kill -0 1234                # test if process exists (no signal sent, just checks)

# Kill by name
killall nginx               # kill all processes named nginx
killall -HUP nginx          # send HUP to all nginx processes
pkill -f "python app.py"    # kill by full command line match
pkill -u alice              # kill all processes owned by alice
pkill -SIGTERM -G developers  # kill all processes owned by developers group

# Find processes
pgrep nginx                 # list PIDs of processes named nginx
pgrep -a nginx              # with full command line
pgrep -u alice              # PIDs owned by alice
pgrep -P 1234               # child PIDs of PPID 1234

# Process info
ps aux                      # all processes, BSD format
# USER  PID  %CPU  %MEM  VSZ  RSS  TTY  STAT  START  TIME  COMMAND

ps -ef                      # all processes, UNIX format (shows PPID)
# UID  PID  PPID  C  STIME  TTY  TIME  CMD

ps aux --sort=-%cpu | head -10   # top CPU consumers
ps aux --sort=-%mem | head -10   # top memory consumers
ps -p 1234 -o pid,ppid,user,%cpu,%mem,cmd  # custom output format

# View process tree
pstree                      # ASCII tree of all processes
pstree -p 1234              # tree rooted at PID 1234
pstree -u                   # show usernames
```

### 🏭 Real-Time Production Use Case

**Scenario**: Production nginx refusing to reload config after certificate update.

```bash
# Check nginx master PID
cat /var/run/nginx.pid        # or: pgrep -o nginx (oldest = master)

# Send SIGHUP for graceful config reload (no connection drops)
kill -HUP $(cat /var/run/nginx.pid)

# Verify workers restarted (new workers = new config applied)
ps aux | grep nginx

# If nginx is stuck in D-state (common with slow NFS for logs):
ps aux | grep " D "           # find D-state processes
cat /proc/$(pgrep nginx)/wchan  # shows kernel function it's waiting in
# Likely: nfs_file_write or similar → fix NFS issue
```

### ❌ Common Mistakes

1. **Using `kill -9` immediately** — SIGKILL prevents any cleanup. Open file handles aren't flushed, temp files aren't removed, database transactions aren't rolled back. Always try `SIGTERM` first, wait 10-30 seconds, then SIGKILL if it doesn't respond.
2. **Ignoring zombie processes** — Zombies themselves consume minimal resources (just a process table entry). But if you have thousands of zombies, you've found a parent process bug (not calling `wait()`). Fix the parent, not the zombie.
3. **Confusing SIGHUP behaviour** — Some programs treat SIGHUP as "reload config" (nginx, sshd, logrotate). Others treat it as "terminal disconnect = quit" (the original meaning). Check the application's documentation — never assume.
4. **`pkill nginx` killing all nginx processes including master** — `pkill` kills ALL matching processes. To kill only workers (and let master restart them), send SIGWINCH to the master. Or use `kill -QUIT` for graceful worker shutdown.

### 🔧 Troubleshooting Tips

```
Problem: Process in D-state (uninterruptible sleep) — cannot be killed
Symptom: ps aux shows "D" state, kill -9 has no effect
Root Cause: Process is waiting in kernel for I/O that isn't completing
Fix:
  cat /proc/<pid>/wchan        # identify blocking kernel function
  cat /proc/<pid>/stack        # full kernel stack trace (requires root)
  # Common causes:
  # nfs_* functions → NFS server unreachable or overloaded
  # ext4_* functions → local disk I/O hang (check dmesg for disk errors)
  # Fix: resolve the underlying I/O issue
  # Nuclear option: reboot (D-state processes cannot be killed)
  dmesg | tail -50             # check for disk errors
  nfsstat                      # check NFS statistics
```

```
Problem: Zombie processes accumulating
Symptom: ps aux shows many "Z" state processes
Root Cause: Parent process not calling wait() after child exits
Fix:
  ps aux | grep "Z"            # find zombies
  ps -p <zombie_pid> -o ppid   # find parent PID
  # Option 1: kill parent (it will adopt zombies to init/systemd which reaps them)
  kill -SIGCHLD <parent_pid>   # signal parent to reap children
  # Option 2: kill parent if it's the bug source
  # Zombies themselves: wait for parent to die or reboot
```

### 🎤 Interview Questions & Answers

**Q: What is a zombie process and how do you handle it?**
A: A **zombie process** is a process that has terminated but whose entry remains in the process table because its parent hasn't called `wait()` to collect its exit status. The process is already dead — it's not consuming CPU or significant memory — just a process table entry. You can't kill a zombie (it's already dead). To handle zombies: find the parent with `ps -p <zombie_pid> -o ppid`, then send `SIGCHLD` to the parent to prompt it to reap children. If that doesn't work, kill the parent process — its zombie children will be inherited by PID 1 (systemd), which will reap them. Many zombies = a bug in the parent application that isn't calling `wait()`.

**Q: What is the difference between SIGTERM and SIGKILL?**
A: **SIGTERM (15)** is a polite termination request. The process can catch this signal and handle it — doing cleanup (closing DB connections, flushing buffers, removing temp files, sending notifications). It's the default signal for `kill`. **SIGKILL (9)** is sent directly to the kernel, which immediately destroys the process. The process never sees it — no cleanup runs, no signal handler executes, file buffers aren't flushed. This can cause data corruption for database processes. Production rule: always try SIGTERM first, wait a reasonable time (systemd's default is 90 seconds), then send SIGKILL only if the process is stuck.

**Q: What does a process in D-state mean and why can't you kill it?**
A: **D-state** (uninterruptible sleep) means the process is waiting inside the kernel for an I/O operation to complete, and the kernel has made that wait uninterruptible. It cannot be interrupted because the kernel needs to maintain consistency — if you killed a process mid-syscall, you could leave kernel data structures in an inconsistent state. SIGKILL doesn't work because signal delivery happens at the process level, but D-state processes are blocked inside kernel code. The only solutions are: (1) resolve the underlying I/O issue (fix the NFS server, replace the failing disk), or (2) reboot. This is why NFS mounts in production should always use the `soft` or `intr` mount options to avoid indefinite D-state hangs.

---

## 15. Process Monitoring & Management

### 📖 Concept Explanation

Real-time process monitoring is essential during incidents. These tools let you see exactly what processes are doing, what resources they're consuming, and what files/sockets they have open — all without restarting anything.

### 🏗️ Architecture Diagram

```
MONITORING TOOLS BY USE CASE:

 "What's using my CPU/RAM?"    "What files/ports are open?"
  ┌──────────────────────┐      ┌──────────────────────────┐
  │  top / htop          │      │  lsof                    │
  │  ps aux --sort=-%cpu │      │  lsof -p <pid>           │
  └──────────────────────┘      │  lsof -i :80             │
                                 └──────────────────────────┘
 "Why is this process slow?"   "What syscalls is it making?"
  ┌──────────────────────┐      ┌──────────────────────────┐
  │  strace -p <pid>     │      │  perf top                │
  │  ltrace -p <pid>     │      │  perf stat -p <pid>      │
  └──────────────────────┘      └──────────────────────────┘

                    "What is the process tree?"
                     ┌──────────────────────┐
                     │  pstree -p           │
                     │  ps -ef (shows PPID) │
                     └──────────────────────┘
```

### ⚙️ Key Commands

```bash
# ─── top ──────────────────────────────────────────────────────
top                     # interactive, updates every 3s
# Key bindings in top:
# P  → sort by CPU usage
# M  → sort by memory usage
# k  → kill process (enter PID)
# r  → renice process
# 1  → toggle per-CPU view
# H  → show threads
# q  → quit

# top header explained:
# load average: 0.52, 0.38, 0.31  → 1min, 5min, 15min averages
# Tasks: 231 total, 1 running, 230 sleeping, 0 stopped, 0 zombie
# %Cpu(s): 5.2 us, 1.3 sy, 0.0 ni, 92.5 id, 0.8 wa, 0.0 hi, 0.2 si
#   us=user, sy=system(kernel), ni=niced, id=idle, wa=iowait, hi=hardware IRQ

# ─── htop ─────────────────────────────────────────────────────
htop                    # improved top: color, mouse support, easy kill
# F5 → tree view  F6 → sort  F9 → kill  F10 → quit
# Particularly useful: F4 to filter by process name

# ─── ps (production queries) ──────────────────────────────────
ps aux                          # all processes, all users
ps aux | grep -v grep | grep nginx   # find nginx, exclude grep line itself
ps -eo pid,ppid,user,pcpu,pmem,comm --sort=-pcpu | head -20  # custom columns, sorted by CPU
ps -p 1234 -o pid,ppid,uid,gid,stat,cmd   # specific PID detail
ps -T -p 1234                   # show all threads of PID 1234
ps aux | awk '$8 ~ /Z/ {print $0}'  # show zombie processes

# ─── pstree ───────────────────────────────────────────────────
pstree                  # full tree from PID 1
pstree -p               # with PIDs
pstree -u               # with usernames
pstree alice            # processes owned by alice

# ─── lsof (List Open Files) ───────────────────────────────────
# In Linux: everything is a file — regular files, sockets, pipes, devices
lsof -p 1234            # all files open by PID 1234
lsof -u alice           # all files open by user alice
lsof /var/log/app.log   # which process has this file open
lsof +D /var/www        # all files open under /var/www (recursive)
lsof -i                 # all network connections (internet files)
lsof -i :80             # what's listening or connected on port 80
lsof -i :80 -i :443     # port 80 and 443
lsof -i TCP:1024-65535  # all high-port TCP connections
lsof -i @192.168.1.100  # connections to specific host
lsof -nP -i TCP         # no hostname/port resolution (faster, numeric)

# Critical use: find what's holding a deleted file open (disk space not freed)
lsof | grep deleted     # files marked deleted but still held open by a process
# Fix: restart the process holding the deleted file, or: kill it

# ─── strace (system call tracer) ──────────────────────────────
strace ls /tmp              # trace all syscalls of a command
strace -p 1234              # attach to running process
strace -p 1234 -e trace=open,read,write  # filter to specific syscalls
strace -p 1234 -e trace=network         # only network syscalls
strace -c -p 1234           # summary count of syscalls (run then Ctrl+C)
strace -f -p 1234           # follow forks (trace child processes too)
strace -o /tmp/trace.log -p 1234  # write trace to file
strace -T -p 1234           # show time spent in each syscall

# ─── ltrace (library call tracer) ─────────────────────────────
ltrace ./myapp              # trace dynamic library calls (malloc, printf, etc.)
ltrace -p 1234              # attach to running process
# Useful when strace shows too much kernel noise — ltrace shows higher-level calls
```

### 🏭 Real-Time Production Use Case

**Scenario**: "Disk is full but `df` shows only 60% used" — classic deleted-file-held-open problem.

```bash
# Step 1: Find the discrepancy
df -h /var        # shows 95% full
du -sh /var/*     # shows only 40% used — gap found!

# Step 2: Find deleted files still open
lsof | grep "deleted" | awk '{print $1, $2, $7, $NF}'
# Output: nginx 1234 /var/log/nginx/access.log (deleted) 45GB

# Step 3: The file was log-rotated but nginx still writing to old (deleted) fd
kill -USR1 $(cat /var/run/nginx.pid)  # signal nginx to reopen log files
# OR: systemctl reload nginx

# Step 4: Verify space is freed
df -h /var        # now shows 60%
```

### ❌ Common Mistakes

1. **Using `strace` on high-traffic production processes** — `strace` has 10-100x overhead. Attaching to a busy nginx worker can cause cascading latency. Use `strace -c` for summary (lower overhead) or use `perf` instead.
2. **Not filtering `lsof` output** — `lsof` without arguments lists thousands of entries. Always filter: `lsof -p <pid>`, `lsof -i :80`, `lsof -u username`.
3. **Misreading `ps` memory columns** — `VSZ` (virtual size) is the total virtual address space — usually much larger than actual RAM used. `RSS` (resident set size) is the actual physical RAM in use. For memory troubleshooting, use RSS, not VSZ.

### 🎤 Interview Questions & Answers

**Q: How do you find which process is listening on port 8080?**
A: Several approaches: `lsof -i :8080` shows all processes with that port open (listening or connected). `ss -tlnp | grep :8080` shows listening sockets with PID and process name. `fuser -n tcp 8080` shows PIDs using the port. `netstat -tlnp | grep :8080` (older but still common). The `ss` approach is fastest and most reliable on modern systems. The output includes the PID and socket state (LISTEN, ESTABLISHED).

**Q: `df -h` shows 90% disk usage but `du -sh /*` shows much less. What's happening?**
A: This is the "deleted files held open" problem. When a process deletes a file while it still has it open (via a file descriptor), the filesystem marks the file as deleted but doesn't release the disk blocks until the last file descriptor is closed. `df` reports actual disk block usage (includes these "deleted but open" files). `du` scans directory entries (which no longer include the deleted file). Fix: `lsof | grep deleted` to find the culprit process, then either restart that process to close its file descriptors, or send the appropriate signal (e.g., `SIGUSR1` to nginx to reopen log files).

---

## 16. systemd & Service Management

### 📖 Concept Explanation

**systemd** is PID 1 on virtually all modern Linux distributions. It replaced SysV init and Upstart. systemd manages the entire system lifecycle: boot, services, logging, timers, mounts, sockets, and more.

**Unit types** (the building blocks of systemd):
- **`.service`**: A daemon or one-shot process (nginx, sshd, your app)
- **`.socket`**: Socket-activated service (starts service when first connection arrives)
- **`.timer`**: Scheduled execution (cron replacement)
- **`.target`**: Group of units / synchronisation point (like runlevels)
- **`.mount`**: Filesystem mount point
- **`.path`**: Path-based activation (start service when file appears)

**Unit file locations** (priority order, highest first):
1. `/etc/systemd/system/` — Admin-created, highest priority, used for overrides
2. `/run/systemd/system/` — Runtime units (cleared on reboot)
3. `/lib/systemd/system/` — Package-installed units (don't modify these directly)

### 🏗️ Architecture Diagram

```
systemd ARCHITECTURE:
┌──────────────────────────────────────────────────────────────────┐
│  PID 1: systemd                                                   │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  default.target (multi-user.target or graphical.target) │     │
│  │  ├── network.target                                      │     │
│  │  │   ├── NetworkManager.service                         │     │
│  │  │   └── systemd-resolved.service                       │     │
│  │  ├── sshd.service                                        │     │
│  │  ├── myapp.service ←── your custom service               │     │
│  │  │   Requires=postgresql.service                        │     │
│  │  │   After=postgresql.service network.target            │     │
│  │  └── cron.service                                        │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                    │
│  ┌──────────────────┐    ┌──────────────────────────────────┐    │
│  │  systemd-journald│    │  systemd-logind                  │    │
│  │  (logging)       │    │  (user session management)       │    │
│  └──────────────────┘    └──────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘

SERVICE STATES:
  active (running)  → service is up and processes are alive
  active (exited)   → one-shot service ran and exited successfully
  failed            → service exited with error or crashed
  inactive (dead)   → service not running, not failed
  activating        → in the process of starting up
  deactivating      → in the process of shutting down
```

### ⚙️ Key Commands

```bash
# ─── Service lifecycle ────────────────────────────────────────
systemctl start nginx           # start now (doesn't survive reboot)
systemctl stop nginx            # stop now
systemctl restart nginx         # stop + start (brief downtime)
systemctl reload nginx          # reload config without restart (if supported)
systemctl reload-or-restart nginx  # use reload if supported, else restart

# ─── Boot-time behaviour ──────────────────────────────────────
systemctl enable nginx          # start at boot (creates symlink in /etc/systemd/system/*.wants/)
systemctl disable nginx         # don't start at boot (removes symlink)
systemctl enable --now nginx    # enable + start immediately (most useful!)
systemctl disable --now nginx   # disable + stop immediately
systemctl mask nginx            # completely prevent start (symlinks to /dev/null)
systemctl unmask nginx          # reverse mask

# ─── Status and inspection ────────────────────────────────────
systemctl status nginx          # status, last 10 log lines, PID, uptime
systemctl is-active nginx       # returns "active" or "inactive" (scriptable)
systemctl is-enabled nginx      # returns "enabled" or "disabled"
systemctl is-failed nginx       # returns "failed" or "active"
systemctl list-units --type=service --state=failed  # list all failed services
systemctl list-units --type=service --state=running # list all running services
systemctl list-unit-files --type=service | grep enabled  # all enabled services
systemctl cat nginx             # show the unit file content
systemctl show nginx            # show all properties (machine-readable)

# ─── Dependencies ─────────────────────────────────────────────
systemctl list-dependencies nginx    # show what nginx depends on
systemctl list-dependencies nginx --reverse  # what depends on nginx

# ─── Daemon reload ────────────────────────────────────────────
systemctl daemon-reload         # reload systemd config (MUST run after editing unit files)

# ─── Boot performance ─────────────────────────────────────────
systemd-analyze                 # total boot time
systemd-analyze blame           # services sorted by startup time
systemd-analyze critical-chain  # critical path (bottleneck chain)

# ─── Writing a custom service unit ────────────────────────────
# /etc/systemd/system/myapp.service
cat << 'EOF' > /etc/systemd/system/myapp.service
[Unit]
Description=MyApp Application Server
Documentation=https://docs.myapp.io
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
EnvironmentFile=/etc/myapp/environment
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5s
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp
# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/myapp /var/log/myapp
PrivateTmp=true
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now myapp

# ─── Override without modifying original unit (drop-in) ───────
# Preferred for package-installed units:
systemctl edit nginx            # creates /etc/systemd/system/nginx.service.d/override.conf
# Example override: add environment variable and change restart policy
mkdir -p /etc/systemd/system/nginx.service.d/
cat << 'EOF' > /etc/systemd/system/nginx.service.d/override.conf
[Service]
Restart=always
RestartSec=3s
Environment=NGINX_CUSTOM_VAR=value
EOF
systemctl daemon-reload

# ─── systemd Timers (cron replacement) ────────────────────────
# Create timer + service pair:
# /etc/systemd/system/backup.service
cat << 'EOF' > /etc/systemd/system/backup.service
[Unit]
Description=Daily Database Backup

[Service]
Type=oneshot
User=backup
ExecStart=/usr/local/bin/backup-db.sh
EOF

# /etc/systemd/system/backup.timer
cat << 'EOF' > /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily at 2AM

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true        # run missed timer if system was off
RandomizedDelaySec=300  # add up to 5min random delay (prevent thundering herd)

[Install]
WantedBy=timers.target
EOF

systemctl enable --now backup.timer
systemctl list-timers         # show all timers and next run time
```

### 🏭 Real-Time Production Use Case

**Scenario**: A Node.js application needs automatic restart, resource limits, and structured logging.

```bash
# /etc/systemd/system/nodeapp.service
[Unit]
Description=Node.js Production App
After=network.target

[Service]
Type=simple
User=nodeapp
WorkingDirectory=/opt/nodeapp
ExecStart=/usr/bin/node /opt/nodeapp/server.js
Restart=always              # restart regardless of exit code
RestartSec=10s              # wait 10s before restarting
StartLimitIntervalSec=60    # in 60 second window
StartLimitBurst=5           # allow max 5 restarts (prevents restart loop)
Environment=NODE_ENV=production PORT=3000
StandardOutput=journal      # logs go to journald
StandardError=journal
SyslogIdentifier=nodeapp
LimitNOFILE=100000          # high file descriptor limit for Node.js
LimitNPROC=1000             # max processes/threads
MemoryMax=2G                # cgroup: hard limit on RAM
CPUQuota=80%                # cgroup: limit to 80% of one CPU core

[Install]
WantedBy=multi-user.target
```

**Result**: Application automatically restarts on crash, resources are bounded (no single app can OOM the node), and all logs flow into journald for centralized collection.

### ❌ Common Mistakes

1. **Editing unit files in `/lib/systemd/system/` directly** — Package upgrades will overwrite your changes. Always use `systemctl edit <service>` for drop-in overrides or copy to `/etc/systemd/system/`.
2. **Forgetting `systemctl daemon-reload`** — After editing ANY unit file, you MUST run `systemctl daemon-reload`. Without it, systemd keeps using the old cached version.
3. **`Type=simple` for services that fork** — If your service forks into the background (`ExecStart=/usr/sbin/nginx` which daemonizes itself), use `Type=forking`. With `Type=simple`, systemd thinks the service is the shell that started it, and when it exits, systemd thinks the service stopped.
4. **Not setting `Restart=on-failure`** — Without this, a crashed service stays dead. Set `Restart=always` for critical services and `Restart=on-failure` for services that may legitimately exit with 0.
5. **Using `ExecStart` with shell pipeline** — `ExecStart=/bin/bash -c "cmd1 | cmd2"` is an antipattern. Use a wrapper script, or use `ExecStartPost` for post-start commands.

### 🔧 Troubleshooting Tips

```
Problem: Service fails to start
Symptom: systemctl status shows "failed" with an error code
Fix:
  systemctl status myapp.service    # check last few log lines
  journalctl -u myapp.service -n 50 # last 50 lines of service journal
  journalctl -u myapp.service --since "10 minutes ago"
  journalctl -xe                    # jump to recent errors with context
  # Check: executable path correct? User exists? EnvironmentFile exists?
  # Try running ExecStart command manually as the service user:
  sudo -u myapp /opt/myapp/bin/server --config /etc/myapp/config.yaml
```

```
Problem: Service keeps restarting in a loop
Symptom: systemctl status shows "activating" or rapid start/fail cycle
Root Cause: ExecStart command fails immediately (wrong path, missing deps)
Fix:
  journalctl -u myapp -f           # watch live as it restarts
  # After too many restarts, systemd stops trying (StartLimitBurst)
  systemctl reset-failed myapp     # clear fail counter to try again
  systemctl start myapp            # try again after fixing the issue
```

### 🎤 Interview Questions & Answers

**Q: What is the difference between `systemctl stop` and `systemctl mask`?**
A: **`systemctl stop`** stops the currently running service instance. It can still be started again manually or will start at next boot if enabled. **`systemctl mask`** creates a symlink from the unit file to `/dev/null`, making it completely impossible to start — by systemd, by another service, or manually. `systemctl start masked-service` returns an error. Use masking when you want to permanently prevent a service from running (e.g., masking `avahi-daemon` or `cups` on a server). `systemctl unmask` reverses it.

**Q: How do you write a systemd service that restarts automatically on failure but doesn't loop infinitely?**
A: Use `Restart=on-failure` (or `Restart=always`), `RestartSec=5s` (cooldown), and the burst limit:
```ini
[Service]
Restart=on-failure
RestartSec=5s
StartLimitIntervalSec=60   # 60 second window
StartLimitBurst=5           # max 5 restart attempts in that window
```
After 5 failures in 60 seconds, systemd stops trying and the service goes into `failed` state. Alert on this with monitoring. To resume: fix the underlying issue, then `systemctl reset-failed myapp && systemctl start myapp`.

**Q: How does systemd's dependency system work? Explain `Wants` vs `Requires` vs `After`.**
A: These are two different dimensions of dependency: **ordering** and **requirement strength**.

`After=postgresql.service` means "start me AFTER postgresql is started, but don't require it to succeed." `Before=` is the inverse.

`Requires=postgresql.service` means "I need postgresql. If postgresql fails to start or stops, stop me too."

`Wants=postgresql.service` is weaker — "I prefer postgresql to be running, but I'll start even if it fails."

Practical pattern: `After=postgresql.service Requires=postgresql.service` — "I need postgres AND must start after it." `After=network.target` (not `Requires`) — "I should start after networking is ready, but start anyway if networking setup fails" (common for services that can handle reconnecting themselves).

---

## 17. Cron & Task Scheduling

### 📖 Concept Explanation

**cron** is the traditional Linux task scheduler. It runs commands at specified times. Despite systemd timers being more powerful, cron remains ubiquitous because of its simplicity and universal availability.

**Crontab format** (5 time fields + command):
```
* * * * * /path/to/command arg1 arg2
│ │ │ │ │
│ │ │ │ └─── Day of week (0-7, 0 and 7 = Sunday)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)

Special strings:
@reboot   → run once at startup
@daily    → 0 0 * * *
@weekly   → 0 0 * * 0
@monthly  → 0 0 1 * *
@hourly   → 0 * * * *
```

### 🏗️ Architecture Diagram

```
CRON ECOSYSTEM:
┌──────────────────────────────────────────────────────────────┐
│                      crond daemon                             │
│  (wakes every minute, checks what needs to run)              │
│                                                               │
│  Reads from:                                                  │
│  ┌─────────────────────┐  ┌────────────────────────────────┐│
│  │ /var/spool/cron/    │  │ /etc/cron.d/                   ││
│  │  username (per-user)│  │  (system jobs with user field) ││
│  └─────────────────────┘  └────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ /etc/crontab (system crontab, has USERNAME field)        │ │
│  └─────────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ /etc/cron.hourly/   (scripts dropped here run hourly) │   │
│  │ /etc/cron.daily/    (scripts run daily by run-parts)  │   │
│  │ /etc/cron.weekly/                                     │   │
│  │ /etc/cron.monthly/                                    │   │
│  └───────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
                          │
            Environment: minimal PATH, no TTY
            Output: emailed to user (if mail set up) or lost!
```

### ⚙️ Key Commands

```bash
# ─── User crontab management ──────────────────────────────────
crontab -e              # edit your crontab (uses $EDITOR)
crontab -l              # list your crontab
crontab -r              # DELETE your entire crontab (dangerous, no confirmation!)
crontab -l > /tmp/crontab.bak && crontab -r  # backup before removing

# Edit another user's crontab (as root)
crontab -u alice -e
crontab -u alice -l

# ─── Cron examples ────────────────────────────────────────────
# Every 5 minutes
*/5 * * * * /usr/local/bin/check-health.sh

# Every day at 2:30 AM
30 2 * * * /usr/local/bin/backup-db.sh

# Weekdays at 8 AM
0 8 * * 1-5 /usr/local/bin/send-daily-report.sh

# First day of every month at midnight
0 0 1 * * /usr/local/bin/monthly-cleanup.sh

# Every 15 minutes during business hours (8AM-6PM, Mon-Fri)
*/15 8-18 * * 1-5 /usr/local/bin/sync-data.sh

# Run at boot
@reboot /usr/local/bin/start-monitoring.sh

# ─── Cron best practices ──────────────────────────────────────
# Always use absolute paths (cron has minimal PATH)
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Always redirect output (or it's emailed / lost)
30 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Avoid overlapping runs with flock
*/5 * * * * flock -n /tmp/mylock /usr/local/bin/check.sh

# ─── System-wide cron (/etc/cron.d/) ──────────────────────────
# /etc/cron.d/ files have an extra USERNAME field:
# m h dom mon dow USER COMMAND
0 3 * * * root /usr/local/bin/system-backup.sh >> /var/log/backup.log 2>&1

# ─── Cron logging ─────────────────────────────────────────────
grep CRON /var/log/syslog | tail -20  # Debian/Ubuntu
grep CRON /var/log/cron | tail -20    # RHEL/CentOS
journalctl -u cron | tail -20         # via journald

# ─── at and batch (one-time scheduling) ───────────────────────
echo "/usr/local/bin/deploy.sh" | at 23:00        # run at 11 PM today
echo "/usr/local/bin/report.sh" | at now + 2 hours # run in 2 hours
at -l                    # list scheduled at jobs
at -d 1                  # delete at job ID 1
atq                      # same as at -l
batch                    # run when system load drops below 1.5

# ─── anacron (for systems not always on) ──────────────────────
# /etc/anacrontab
# PERIOD  DELAY  JOB-ID  COMMAND
# period in days, delay in minutes before starting
1   5   daily-jobs   run-parts /etc/cron.daily
7   10  weekly-jobs  run-parts /etc/cron.weekly
```

### 🏭 Real-Time Production Use Case

**Scenario**: A cron job that was working correctly suddenly stops running silently.

```bash
# Step 1: Check if cron is running
systemctl status cron   # or crond on RHEL

# Step 2: Check cron logs
grep "backup.sh" /var/log/syslog | tail -20
# Look for: "CMD (/usr/local/bin/backup.sh)" - confirms cron launched it

# Step 3: Test the command manually as the cron user
sudo -u root env -i PATH=/usr/bin:/bin /usr/local/bin/backup.sh
# env -i clears environment (simulates cron's clean environment)

# Step 4: Common fix - PATH issue
# Add at top of crontab:
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Step 5: Ensure output is captured
30 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

### ❌ Common Mistakes

1. **Relative paths in cron** — Cron's `$PATH` is typically just `/usr/bin:/bin`. Always use full absolute paths for both the command AND any files it references.
2. **No output redirection** — By default, cron emails output to the user. If no mail is configured, output is silently discarded. Always: `command >> /var/log/myjob.log 2>&1`.
3. **`crontab -r` instead of `crontab -e`** — Easy typo that deletes the entire crontab with no confirmation. Always back up first: `crontab -l > ~/crontab.bak`.
4. **Not preventing overlapping runs** — If a cron job takes longer than its schedule interval, you'll have multiple instances running simultaneously. Use `flock -n /tmp/job.lock /path/to/script` to prevent this.
5. **Environment variables not available** — Cron doesn't load `.bashrc` or `.bash_profile`. Variables like `$JAVA_HOME`, `$NVM_DIR`, `$GOPATH` won't be available. Set them explicitly in the crontab or script.

### 🎤 Interview Questions & Answers

**Q: A cron job works when run manually but fails silently when run by cron. How do you debug this?**
A: This is the classic cron debugging scenario. The two most common causes are: (1) **PATH issue** — cron runs with a minimal `PATH`. Test by running the command with cron's environment: `sudo -u <user> env -i PATH=/usr/bin:/bin HOME=/root /usr/local/bin/myscript.sh`. (2) **Missing output capture** — add `>> /tmp/job.log 2>&1` to capture all output. For thorough debugging: add `set -x` at the top of the script, check `/var/log/syslog` for the cron execution entry, verify the script has execute permission (`ls -la /path/to/script`), and confirm the user's crontab is syntactically valid with `crontab -l`.

**Q: When would you use systemd timers instead of cron? What are the advantages?**
A: **systemd timers** are preferable when: (1) You need the timer to run a missed execution after a system was off (`Persistent=true`) — anacron does this for cron, but it's messier; (2) You need the timer job to have resource limits (CPU, memory via cgroups); (3) You want job logs in journald alongside all other service logs; (4) You need calendar-based expressions beyond cron's capabilities (e.g., "last Friday of the month"); (5) The timer is tightly coupled to a systemd service. **Cron advantages**: universally available, simpler for basic scheduling, familiar to everyone, works without systemd (containers, minimal systems). In practice: use systemd timers for new infrastructure on systemd-based systems; keep cron for portability and simplicity.

---

# 🟣 SECTION 5 — PACKAGE MANAGEMENT

---

## 18. Package Management (apt / yum / dnf)

### 📖 Concept Explanation

Package managers automate the installation, upgrade, configuration, and removal of software. They handle dependency resolution (if package A requires B and C, install those too) and maintain a database of installed packages.

**Two major ecosystems**:
- **`.deb` + apt**: Debian, Ubuntu, Kali, Raspberry Pi OS
- **`.rpm` + dnf/yum**: RHEL, CentOS, Rocky, AlmaLinux, Fedora, Amazon Linux, SUSE

| Command Purpose | apt (Debian/Ubuntu) | dnf (RHEL/Fedora) |
|---|---|---|
| Install package | `apt install nginx` | `dnf install nginx` |
| Remove package | `apt remove nginx` | `dnf remove nginx` |
| Remove + configs | `apt purge nginx` | `dnf remove nginx` (configs may remain) |
| Update index | `apt update` | `dnf check-update` |
| Upgrade all | `apt upgrade` | `dnf upgrade` |
| Full upgrade | `apt full-upgrade` | `dnf distro-sync` |
| Search | `apt search nginx` | `dnf search nginx` |
| Package info | `apt show nginx` | `dnf info nginx` |
| List installed | `dpkg -l` | `rpm -qa` |
| What provides file | `apt-file search /bin/ls` | `dnf provides /bin/ls` |
| Clean cache | `apt clean` | `dnf clean all` |

### 🏗️ Architecture Diagram

```
PACKAGE MANAGER ARCHITECTURE:
┌─────────────────────────────────────────────────────────────────┐
│  USER: apt install nginx                                         │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  APT/DNF (High-Level Package Manager)                    │   │
│  │  • Dependency resolution                                  │   │
│  │  • Repository management                                  │   │
│  │  • Download management                                    │   │
│  └─────────────────────┬───────────────────────────────────┘   │
│                         │ calls                                  │
│  ┌─────────────────────▼───────────────────────────────────┐   │
│  │  dpkg / rpm (Low-Level Package Manager)                  │   │
│  │  • Actually install/remove .deb/.rpm files               │   │
│  │  • Maintain installed package database                    │   │
│  │  • Run pre/post install scripts                          │   │
│  └─────────────────────┬───────────────────────────────────┘   │
│                         │                                        │
│  ┌─────────────────────▼───────────────────────────────────┐   │
│  │  Package Database                                         │   │
│  │  /var/lib/dpkg/status (Debian)                           │   │
│  │  /var/lib/rpm/Packages (RHEL)                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  REMOTE: Repository Servers                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│  │ Ubuntu Main  │  │ Ubuntu Univ. │  │ PPA / Custom Repo  │   │
│  │ archive.u.c  │  │ archive.u.c  │  │ ppa:nginx/stable   │   │
│  └──────────────┘  └──────────────┘  └────────────────────┘   │
│  Config: /etc/apt/sources.list, /etc/apt/sources.list.d/        │
│  RHEL:   /etc/yum.repos.d/*.repo                               │
└─────────────────────────────────────────────────────────────────┘
```

### ⚙️ Key Commands

```bash
# ══════════════════════════════════════════
# ── apt (Debian / Ubuntu) ─────────────────
# ══════════════════════════════════════════

# Update package index (ALWAYS do this before install)
apt update

# Install packages
apt install -y nginx curl git     # -y: non-interactive (no prompts)
apt install -y nginx=1.24.0       # install specific version

# Upgrade
apt upgrade -y                    # upgrade all, don't remove packages
apt full-upgrade -y               # upgrade, may remove conflicting packages
apt-get dist-upgrade              # older syntax, same as full-upgrade

# Remove
apt remove nginx                  # remove binary, keep config files
apt purge nginx                   # remove binary + config files
apt autoremove                    # remove orphaned dependencies

# Search and inspect
apt search "http server"          # search by description
apt show nginx                    # detailed package info
apt list --installed | grep nginx # is nginx installed?
apt list --upgradable             # packages with available updates

# Repository management
add-apt-repository ppa:nginx/stable           # add PPA
add-apt-repository "deb https://nginx.org/packages/ubuntu focal nginx"
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys KEY_ID
# Modern way (no apt-key):
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
  | tee /usr/share/keyrings/nginx-archive-keyring.gpg > /dev/null

# Hold/pin package at specific version (prevent upgrade)
apt-mark hold nginx               # hold at current version
apt-mark unhold nginx             # remove hold
apt-mark showhold                 # list held packages

# ─── dpkg (low level) ─────────────────────────────────────────
dpkg -l                           # list all installed packages
dpkg -l nginx                     # check if nginx installed
dpkg -L nginx                     # list files installed by package
dpkg -S /usr/sbin/nginx           # which package owns this file
dpkg -i package.deb               # install a .deb file directly
dpkg -r nginx                     # remove package (keep config)
dpkg -P nginx                     # purge package (remove config too)
dpkg --get-selections | grep -v deinstall  # list installed
dpkg-query -W -f='${Package}\t${Version}\n' | sort  # package list with versions

# ══════════════════════════════════════════
# ── dnf (RHEL 8+ / Fedora / CentOS Stream)
# ══════════════════════════════════════════
dnf install -y nginx
dnf update -y                     # update all packages
dnf update -y nginx               # update specific package
dnf remove nginx
dnf search nginx
dnf info nginx
dnf list installed
dnf list available | grep nginx
dnf provides /usr/sbin/nginx      # what package provides this file
dnf repolist                      # list configured repositories
dnf repolist all                  # including disabled repos
dnf history                       # list all dnf transactions
dnf history undo 5                # undo transaction #5 (rollback!)
dnf group install "Development Tools"  # install package group
dnf clean all                     # clear all caches

# Version locking (dnf)
dnf install 'dnf-command(versionlock)'
dnf versionlock add nginx         # lock nginx at current version
dnf versionlock list
dnf versionlock delete nginx

# ─── rpm (low level) ──────────────────────────────────────────
rpm -qa                           # list all installed RPMs
rpm -qa | grep nginx
rpm -ql nginx                     # files installed by nginx package
rpm -qf /usr/sbin/nginx           # which package owns this file
rpm -qi nginx                     # package info
rpm -ivh package.rpm              # install .rpm file (-v verbose, -h progress)
rpm -Uvh package.rpm              # upgrade .rpm file
rpm -e nginx                      # erase/remove package
rpm --verify nginx                # verify package integrity (detect tampering)
rpm -Va                           # verify ALL installed packages (security audit)

# ─── Repository configuration ─────────────────────────────────
# Add a repo in RHEL/CentOS:
cat << 'EOF' > /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF
dnf install nginx
```

### 🏭 Real-Time Production Use Case

**Scenario**: Auditing installed packages on 200 servers for security compliance.

```bash
#!/bin/bash
# Run on each server via Ansible or parallel SSH
# List packages with versions, sorted — feed to diff to find server inconsistencies

# Debian/Ubuntu:
dpkg-query -W -f='${Package}\t${Version}\t${Architecture}\n' | sort > /tmp/packages_$(hostname).txt

# RHEL/CentOS:
rpm -qa --queryformat '%{NAME}\t%{VERSION}-%{RELEASE}\t%{ARCH}\n' | sort > /tmp/packages_$(hostname).txt

# Compare two servers:
diff /tmp/packages_server1.txt /tmp/packages_server2.txt

# Find packages with known vulnerabilities (using trivy or similar)
# Or: check for packages older than security threshold
apt list --upgradable 2>/dev/null | grep -c "upgradable"  # count pending security updates
```

### ❌ Common Mistakes

1. **Running `apt upgrade` without `apt update` first** — `apt upgrade` works from a local cache. If the cache is stale (days/weeks old), you're not getting the latest available versions. Always `apt update && apt upgrade`.
2. **`apt remove` vs `apt purge`** — `apt remove` leaves configuration files behind. For a clean reinstall or complete removal, use `apt purge`. For containers, always `apt purge` to keep images small.
3. **Mixing package sources** — Adding multiple PPAs or third-party repos for the same package (e.g., nginx from Ubuntu repo AND nginx.org repo) causes conflicts and unpredictable upgrades. Use `apt-cache policy nginx` to see which repo a package comes from.
4. **Not pinning versions in production** — `apt install nginx` installs the latest available. A routine `apt upgrade` can silently change your nginx from 1.24 to 1.26, potentially breaking things. Pin critical packages.
5. **Installing build tools in production containers** — `gcc`, `make`, `git` etc. in a production image increase attack surface and image size. Use multi-stage Docker builds: build in a build image, copy artifact to a minimal runtime image.

### 🎤 Interview Questions & Answers

**Q: What is the difference between `apt remove` and `apt purge`?**
A: **`apt remove`** uninstalls the package binary and associated files, but **keeps configuration files** (typically in `/etc/`). This is useful if you plan to reinstall the same package later and want to preserve your configuration. **`apt purge`** (or `apt-get purge`) removes both the binary and all associated configuration files. Use purge when you want a complete clean removal — for example, before reinstalling with fresh default config, or when decommissioning a service. The equivalent in rpm/dnf is just `dnf remove`, which by default removes the package but may leave config files with `.rpmsave` extensions.

**Q: How do you find which package provides a specific file on Debian vs RHEL?**
A: For a **file already on the system** — on Debian: `dpkg -S /usr/bin/curl`. On RHEL: `rpm -qf /usr/bin/curl`. For a **file not yet installed** — on Debian: `apt-file search curl` (requires `apt-file` package and `apt-file update`). On RHEL: `dnf provides /usr/bin/curl` or `dnf provides curl`. This is extremely useful when a program says "command not found" and you need to know which package to install.

---

## 19. Compiling from Source

### 📖 Concept Explanation

Sometimes you need a version not in any repository, need to enable custom compile-time options, or need to apply patches. Compiling from source is the solution — but it comes with a management overhead cost.

**Standard build flow**:
1. **`./configure`** — Checks dependencies, detects system features, generates Makefile
2. **`make`** — Compiles source code using generated Makefile
3. **`make install`** — Installs compiled files to system (default: `/usr/local/`)

### ⚙️ Key Commands

```bash
# ─── Install build toolchain ──────────────────────────────────
# Debian/Ubuntu
apt install -y build-essential cmake pkg-config libssl-dev zlib1g-dev

# RHEL/CentOS
dnf groupinstall -y "Development Tools"
dnf install -y cmake openssl-devel zlib-devel

# ─── Standard source compilation ──────────────────────────────
# 1. Download and extract
wget https://example.com/app-2.4.tar.gz
tar -xzf app-2.4.tar.gz
cd app-2.4/

# 2. Read build instructions
cat README
cat INSTALL

# 3. Configure (check --help for all options)
./configure --prefix=/usr/local \
            --with-ssl \
            --without-debug \
            --enable-threads

# If configure fails: read the error — usually a missing -dev/-devel package
# Example: "ssl/ssl.h not found" → apt install libssl-dev

# 4. Compile (use -j for parallel — faster on multi-core)
make -j$(nproc)    # nproc = number of CPU cores

# 5. (Optional) Test before installing
make test
make check

# 6. Install
sudo make install  # installs to /usr/local/ by default

# ─── Verify installation ──────────────────────────────────────
which myapp                    # should show /usr/local/bin/myapp
ldconfig                       # update shared library cache (run after installing libs)
ldconfig -p | grep mylib       # check if new library is registered

# ─── cmake-based projects ─────────────────────────────────────
mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
sudo make install

# ─── Uninstalling manually compiled software ──────────────────
# If Makefile has uninstall target:
sudo make uninstall

# If not: track with checkinstall (creates a .deb or .rpm from make install)
apt install checkinstall
sudo checkinstall    # replaces `make install`, creates installable package
dpkg -r your-package # now uninstallable like any deb package

# ─── Managing manual installs ─────────────────────────────────
# Best practice: install to /opt/appname/ to isolate
./configure --prefix=/opt/nginx-custom
# Create versioned directories: /opt/nginx-1.25.3/
# Symlink: ln -sf /opt/nginx-1.25.3 /opt/nginx-current
```

### 🏭 Real-Time Production Use Case

**Scenario**: Need nginx compiled with custom modules (ngx_brotli compression, ModSecurity WAF) not available in the distribution package.

```bash
# Compile nginx with brotli support
apt install -y libpcre3-dev libssl-dev zlib1g-dev
git clone --recurse-submodules https://github.com/google/ngx_brotli /opt/ngx_brotli

wget http://nginx.org/download/nginx-1.25.3.tar.gz
tar -xzf nginx-1.25.3.tar.gz
cd nginx-1.25.3/

./configure \
  --prefix=/etc/nginx \
  --sbin-path=/usr/sbin/nginx \
  --with-http_ssl_module \
  --with-http_v2_module \
  --add-module=/opt/ngx_brotli

make -j$(nproc)
make install

# Now wrap in systemd service + manage upgrades carefully
```

### 🎤 Interview Questions & Answers

**Q: When would you compile from source vs use a package manager? What are the trade-offs?**
A: Use **package manager** (strongly preferred) when: a maintained package exists, the version is acceptable, and you don't need custom compile flags. Package manager gives you: automatic security updates, dependency management, clean uninstall, version tracking. Compile from source when: (1) the required version isn't in any repository; (2) you need custom compile flags or modules not in the packaged version; (3) you need the absolute latest for a critical security fix before it hits the repo; (4) patching the source yourself. Trade-offs of source compilation: you own all upgrades (no `apt upgrade` covering it), dependency hell (system libs can conflict), no clean uninstall unless you track files. Mitigate with `checkinstall` (wraps compiled install into a .deb/.rpm) or install to `/opt/` with a dedicated prefix.

---

# 🟡 SECTION 6 — NETWORKING

---

## 20. Linux Networking Fundamentals

### 📖 Concept Explanation

Linux networking is built in layers following the OSI model, with tools and configuration files at each layer. The key configuration files are:

- **`/etc/hosts`**: Static hostname-to-IP mapping (checked before DNS)
- **`/etc/resolv.conf`**: DNS resolver configuration (nameservers, search domains)
- **`/etc/nsswitch.conf`**: Name Service Switch — defines order of resolution (files, dns, mDNS)

The **`ip` command** is the modern replacement for `ifconfig`, `route`, and `arp`. Always use `ip` in scripts — `ifconfig` is deprecated and not installed by default on many modern systems.

### 🏗️ Architecture Diagram

```
OSI MODEL → LINUX NETWORKING TOOLS:

 Layer 7: Application  │ curl, wget, nc, telnet, nmap
 ─────────────────────────────────────────────────────
 Layer 4: Transport    │ ss, netstat (TCP/UDP ports)
 ─────────────────────────────────────────────────────
 Layer 3: Network      │ ip route, traceroute, ping
                       │ iptables/nftables (routing)
 ─────────────────────────────────────────────────────
 Layer 2: Data Link    │ ip link, arp, ip neigh
 ─────────────────────────────────────────────────────
 Layer 1: Physical     │ ethtool, mii-tool

NAME RESOLUTION FLOW (nsswitch.conf):
  hostname lookup
       │
       ▼
  /etc/nsswitch.conf: "hosts: files dns"
       │
       ├── files → check /etc/hosts first
       │         (instant, no network needed)
       │         127.0.0.1  localhost
       │         192.168.1.10  db.internal
       │
       └── dns → query servers in /etc/resolv.conf
                nameserver 8.8.8.8
                nameserver 8.8.4.4
                search mycompany.internal

NETWORK INTERFACE NAMING (systemd predictable names):
  eth0  → old naming (deprecated)
  ens3  → embedded NIC, slot 3 (VM typical)
  enp3s0 → PCI NIC, bus 3, slot 0
  eno1  → onboard NIC 1
  bond0 → bonded interface
  veth0 → virtual ethernet (containers)
  docker0 → Docker bridge interface
  cni0  → Kubernetes CNI bridge
```

### ⚙️ Key Commands

```bash
# ─── Interface inspection ─────────────────────────────────────
ip addr show                    # all interfaces with IP addresses
ip addr show eth0               # specific interface
ip -4 addr show                 # IPv4 only
ip -6 addr show                 # IPv6 only
ip link show                    # link layer info (MAC, state, MTU)
ip -s link show eth0            # with statistics (RX/TX packets, errors)
ethtool eth0                    # detailed NIC info (speed, duplex, firmware)
ethtool -i eth0                 # driver information

# ─── Routing table ────────────────────────────────────────────
ip route show                   # routing table
ip route show table all         # all routing tables
ip route get 8.8.8.8            # which route would be used for this destination
ip route add 10.0.0.0/8 via 192.168.1.1 dev eth0   # add static route
ip route add default via 192.168.1.1                # add default gateway
ip route del 10.0.0.0/8                            # delete route
# Persistent routes: add to /etc/network/interfaces (Debian) or nmcli (RHEL)

# ─── ARP table ────────────────────────────────────────────────
ip neigh show                   # ARP table (IP → MAC mappings)
ip neigh flush all              # flush ARP cache
arp -n                          # legacy arp command (still available)

# ─── Temporary IP configuration ───────────────────────────────
# Add IP (non-persistent — lost on reboot or NetworkManager restart)
ip addr add 192.168.1.100/24 dev eth0
ip addr del 192.168.1.100/24 dev eth0
ip link set eth0 up             # bring interface up
ip link set eth0 down           # bring interface down
ip link set eth0 mtu 9000       # set jumbo frames (MTU 9000)
ip link set eth0 promisc on     # promiscuous mode (needed for packet capture)

# ─── /etc/hosts editing ───────────────────────────────────────
# Quick override for testing (before DNS propagation):
echo "192.168.1.50 app.prod.internal" >> /etc/hosts

# ─── DNS configuration ────────────────────────────────────────
cat /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 8.8.4.4
# search mycompany.internal ec2.internal

# On systemd-resolved systems (Ubuntu 18.04+):
resolvectl status              # show DNS configuration per interface
resolvectl query google.com    # query DNS
systemd-resolve --status
cat /etc/resolv.conf           # may be symlink to systemd-resolved's stub
```

### 🏭 Real-Time Production Use Case

**Scenario**: Container host with inconsistent DNS resolution — some containers can resolve, others can't.

```bash
# Check host DNS config
cat /etc/resolv.conf
resolvectl status

# Check systemd-resolved is running
systemctl status systemd-resolved

# Docker uses host's /etc/resolv.conf as basis for container DNS
# If /etc/resolv.conf points to 127.0.0.53 (systemd-resolved),
# Docker may not forward correctly to containers
# Fix: either use explicit DNS in Docker daemon config:
cat /etc/docker/daemon.json
# {"dns": ["8.8.8.8", "8.8.4.4"]}
# OR: ensure systemd-resolved is configured correctly
```

---

## 21. Network Configuration & Tools

### 📖 Concept Explanation

Modern Linux uses **NetworkManager** (on desktop and server distros) or **systemd-networkd** (on minimal/server installs) for network configuration. Ubuntu 17.10+ introduced **netplan** as a unified frontend that generates config for either backend.

### 🏗️ Architecture Diagram

```
NETWORK CONFIGURATION STACK:

  ┌──────────────────────────────────────────────────────────┐
  │  CONFIGURATION LAYER                                      │
  │  netplan (/etc/netplan/*.yaml)   ← Ubuntu 18.04+         │
  │  nmcli/nmtui                     ← RHEL, Ubuntu desktop  │
  │  /etc/network/interfaces         ← Debian, older Ubuntu  │
  └────────────────────┬─────────────────────────────────────┘
                        │ generates config for
           ┌────────────┴────────────┐
           ▼                          ▼
  ┌──────────────────┐   ┌─────────────────────────┐
  │  NetworkManager  │   │  systemd-networkd        │
  │  (desktop/server)│   │  (server/minimal)        │
  └──────────────────┘   └─────────────────────────┘
           │                          │
           └────────────┬─────────────┘
                        ▼
              ┌──────────────────┐
              │  Kernel Network  │
              │  Stack           │
              └──────────────────┘
```

### ⚙️ Key Commands

```bash
# ─── nmcli (NetworkManager CLI) ───────────────────────────────
nmcli device status             # show all network devices
nmcli connection show           # show all connections (profiles)
nmcli connection show --active  # active connections only
nmcli device show eth0          # detailed info for eth0

# Set static IP
nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses "192.168.1.100/24" \
  ipv4.gateway "192.168.1.1" \
  ipv4.dns "8.8.8.8,8.8.4.4"
nmcli connection up "Wired connection 1"

# Set DHCP
nmcli connection modify eth0 ipv4.method auto
nmcli connection up eth0

# Add DNS server without changing other settings
nmcli connection modify eth0 ipv4.dns "1.1.1.1 8.8.8.8"
nmcli connection reload

# ─── netplan (Ubuntu 18.04+) ──────────────────────────────────
# /etc/netplan/01-network-manager-all.yaml
cat << 'EOF' > /etc/netplan/01-static.yaml
network:
  version: 2
  renderer: networkd    # or: NetworkManager
  ethernets:
    ens3:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        search: [mycompany.internal]
        addresses: [8.8.8.8, 8.8.4.4]
EOF
netplan try       # apply and auto-revert after 120s if no confirmation
netplan apply     # apply permanently

# ─── Bonding (NIC redundancy/aggregation) ─────────────────────
# Mode 1 (active-backup): failover — one active, one standby
# Mode 4 (802.3ad/LACP): requires switch support, full bandwidth aggregation
# Via nmcli:
nmcli connection add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup"
nmcli connection add type ethernet slave-type bond con-name bond0-port1 ifname eth0 master bond0
nmcli connection add type ethernet slave-type bond con-name bond0-port2 ifname eth1 master bond0
nmcli connection up bond0

# Check bond status
cat /proc/net/bonding/bond0
```

---

## 22. Network Diagnostics & Troubleshooting

### 📖 Concept Explanation

Network troubleshooting in production requires a systematic, layer-by-layer approach. You must rule out each layer before moving to the next.

### 🏗️ Architecture Diagram

```
NETWORK TROUBLESHOOTING FLOWCHART:

  "Cannot reach server X"
           │
           ▼
  ┌────────────────────┐    FAIL    ┌───────────────────────────────┐
  │ 1. ping -c4 X      │──────────▶│ DNS? → dig X / nslookup X     │
  └────────────────────┘           │ Route? → ip route get X.IP     │
           │ PASS                  │ Firewall? → iptables -L        │
           ▼                       └───────────────────────────────┘
  ┌────────────────────┐    FAIL    ┌───────────────────────────────┐
  │ 2. ping gateway    │──────────▶│ Check local NIC, cable, VLAN  │
  └────────────────────┘           └───────────────────────────────┘
           │ PASS
           ▼
  ┌────────────────────┐    FAIL    ┌───────────────────────────────┐
  │ 3. traceroute X    │──────────▶│ Find where packets stop       │
  │    or mtr X        │           │ MTU issue? Routing loop?      │
  └────────────────────┘           └───────────────────────────────┘
           │ PASS
           ▼
  ┌────────────────────┐    FAIL    ┌───────────────────────────────┐
  │ 4. nc -zv X PORT   │──────────▶│ Port not open → is service up?│
  │    curl X:PORT     │           │ Firewall blocking → iptables  │
  └────────────────────┘           └───────────────────────────────┘
           │ PASS
           ▼
  Application-level issue
```

### ⚙️ Key Commands

```bash
# ─── Connectivity testing ─────────────────────────────────────
ping -c 4 8.8.8.8           # ICMP ping (4 packets)
ping -c 4 -s 1472 8.8.8.8  # test MTU (1472 + 28 byte ICMP header = 1500 byte MTU)
ping6 ::1                   # IPv6 ping

# ─── Path tracing ─────────────────────────────────────────────
traceroute 8.8.8.8          # trace route (UDP by default)
traceroute -T -p 443 8.8.8.8  # TCP traceroute (useful when UDP blocked)
traceroute -I 8.8.8.8       # ICMP traceroute
tracepath 8.8.8.8           # similar, doesn't need root

# mtr (combines ping + traceroute, real-time statistics)
mtr 8.8.8.8                 # interactive real-time
mtr --report --report-cycles 20 8.8.8.8  # 20-packet report (great for support tickets)
mtr -n --report 8.8.8.8    # no DNS resolution (faster)

# ─── Socket / Port inspection ─────────────────────────────────
# ss (replaces netstat — modern, faster)
ss -tlnp            # TCP + Listening + numeric + with Process
ss -ulnp            # UDP + Listening + numeric + with Process
ss -anp | grep :80  # all sockets on port 80
ss -tp              # established TCP connections with process
ss -s               # socket statistics summary
ss -tnp state established  # only established connections
ss -tnp 'dst :443'  # connections to port 443

# netstat (legacy but still ubiquitous)
netstat -tlnp       # same as ss -tlnp
netstat -an | grep ESTABLISHED | wc -l  # count established connections

# ─── Port testing ─────────────────────────────────────────────
nc -zv 192.168.1.100 80     # test TCP port (z=zero-I/O, v=verbose)
nc -zv -w 3 host 443        # with 3 second timeout
nc -zvu host 53             # UDP port test
nmap -p 80,443 192.168.1.100  # scan specific ports
nmap -sV -p 22 host         # detect service version on port

# HTTP testing
curl -v http://example.com               # verbose (shows headers)
curl -I http://example.com               # HEAD request (headers only)
curl -sk https://example.com            # skip SSL verification (testing)
curl -o /dev/null -s -w "%{http_code} %{time_total}\n" http://example.com  # timing test
curl --resolve example.com:443:10.0.0.1 https://example.com  # force DNS to specific IP
wget -q -O /dev/null --server-response http://example.com 2>&1 | grep "HTTP/"

# ─── Packet capture ───────────────────────────────────────────
tcpdump -i eth0                       # capture all on eth0
tcpdump -i eth0 port 80               # only HTTP traffic
tcpdump -i eth0 host 192.168.1.100    # traffic to/from specific host
tcpdump -i eth0 'tcp port 80 and host 192.168.1.100'  # combined filter
tcpdump -i eth0 -w /tmp/capture.pcap  # write to file (open in Wireshark)
tcpdump -i eth0 -c 1000 -w /tmp/capture.pcap  # capture 1000 packets then stop
tcpdump -i eth0 -n -X port 53        # show raw hex/ASCII, no DNS resolution

# tshark (Wireshark CLI)
tshark -i eth0 -Y "http.request" -T fields -e http.host -e http.request.uri
# ^ show all HTTP requests with host and URI

# ─── nc (netcat) — Swiss Army Knife ──────────────────────────
# Chat / port test
nc -l 1234                  # listen on port 1234
nc host 1234                # connect to listening nc

# File transfer
nc -l 1234 > received_file  # receive on server
nc server 1234 < file_to_send  # send from client

# Simple HTTP server test
echo -e "HTTP/1.1 200 OK\n\nHello" | nc -l 8080

# Reverse shell (FOR UNDERSTANDING/SECURITY TESTING ONLY)
# nc -l 4444 -e /bin/bash   # listener (attacker)
# nc attacker 4444 -e /bin/bash  # victim

# Port scanning (never do on systems you don't own)
nc -z -v host 20-30   # scan ports 20-30
```

### 🏭 Real-Time Production Use Case

**Scenario**: Intermittent packet loss to a production database server at specific times.

```bash
# Run mtr in background for 30 minutes to capture the issue:
mtr --report --report-cycles 1800 --interval 1 db.prod.internal > /tmp/mtr_report.txt &

# While that runs, capture packets during the next incident:
tcpdump -i eth0 host db.prod.internal -w /tmp/db_capture.pcap -c 50000

# After incident: analyze the mtr report
cat /tmp/mtr_report.txt
# Look for: hops with > 1% loss (especially if only one hop shows loss = that hop is the problem)
# Distinguish: all downstream hops show loss = upstream problem (not destination)
#              only last hop shows loss = destination's ICMP rate limiting (often benign)

# Check TCP retransmits during incident:
ss -tin | grep db.prod.internal  # shows TCP retransmit counters per connection
```

### ❌ Common Mistakes

1. **Concluding "network is down" from ping alone** — ICMP (ping) can be blocked by firewalls while TCP/UDP works fine. Always test the actual protocol/port: `nc -zv host 5432` for PostgreSQL, `curl http://host:80` for HTTP.
2. **Using `netstat` in modern scripts** — `netstat` is deprecated and not installed by default in many systems (it's in `net-tools` package). Use `ss` which is faster, more accurate, and always available.
3. **Interpreting `traceroute` asterisks as packet loss** — `***` in traceroute means that hop didn't respond to probe packets, not that packets are lost there. The route may still work fine. Use `mtr` with report mode for accurate loss statistics.
4. **Running `tcpdump` without limiting capture** — `tcpdump -i eth0` without filters on a busy interface generates GB of data quickly. Always filter by port/host and use `-c` to limit packet count or `-w` with rotation.

### 🔧 Troubleshooting Tips

```
Problem: Cannot reach external host, but local network works
Symptom: ping gateway succeeds, ping 8.8.8.8 fails
Fix:
  ip route show default     # check default gateway is set
  iptables -L FORWARD       # check forwarding rules
  cat /proc/sys/net/ipv4/ip_forward  # is IP forwarding enabled (for routers/VMs)?
  # Check if DNS is the issue:
  ping 8.8.8.8              # using IP (bypasses DNS)
  ping google.com           # using hostname (needs DNS)
  # If IP works but hostname fails → DNS issue
  cat /etc/resolv.conf      # check nameserver
  dig @8.8.8.8 google.com  # test with specific nameserver
```

```
Problem: "Connection refused" on port 8080
Symptom: nc -zv host 8080 returns "Connection refused"
Fix:
  # On the server:
  ss -tlnp | grep :8080    # is anything listening? (if empty, nothing on that port)
  systemctl status myapp   # is the service running?
  # If listening but still refused from remote:
  iptables -L INPUT -n -v | grep 8080   # check INPUT chain
  firewall-cmd --list-all               # firewalld
  ufw status                            # ufw
```

### 🎤 Interview Questions & Answers

**Q: What is the difference between `ss` and `netstat`? Why should you prefer `ss`?**
A: Both show socket/network connection information, but `ss` (Socket Statistics) is the modern replacement from the `iproute2` package. `netstat` is from the deprecated `net-tools` package and isn't installed by default on many modern systems. Key differences: `ss` is significantly faster (reads directly from kernel netlink socket rather than scanning `/proc/net/`), shows more information (TCP internal state, timer info with `-t -i`), and has more powerful filtering. The output format is similar: `ss -tlnp` vs `netstat -tlnp`. In scripts written for modern systems, always use `ss`.

**Q: How would you diagnose and identify a suspected packet loss issue between two servers?**
A: My approach: (1) **`mtr --report --report-cycles 100 destination`** — gives a combined traceroute and ping report showing per-hop loss %. Key insight: if only the last hop shows loss, it's often ICMP rate limiting on the destination (benign). If an intermediate hop shows loss AND all subsequent hops also show higher loss, that hop is the problem. (2) **`tcpdump -i eth0 host destination -w capture.pcap`** — capture actual traffic during the incident. Look for TCP retransmissions (duplicate ACKs, RST packets). (3) **`ss -tin | grep destination`** — shows TCP retransmit counters per connection, confirming packet loss at TCP level. (4) Check NIC statistics: `ip -s link show eth0` for RX/TX errors and drops — hardware-level drops indicate a NIC, cable, or switch port problem.

---

## 23. Firewall (iptables & nftables & firewalld)

### 📖 Concept Explanation

Linux firewall implementation:
- **iptables**: The classic netfilter userspace tool (kernel 2.4+). Still dominant but being replaced.
- **nftables**: Modern replacement for iptables (kernel 3.13+, default on RHEL 8+, Debian 10+).
- **firewalld**: A management daemon using zones, sitting on top of iptables or nftables.
- **ufw** (Uncomplicated Firewall): Simplified iptables frontend, default on Ubuntu.

**iptables architecture** — packets pass through **chains** in **tables**:

| Table | Chains | Purpose |
|---|---|---|
| **filter** | INPUT, OUTPUT, FORWARD | Packet filtering (allow/drop) |
| **nat** | PREROUTING, OUTPUT, POSTROUTING | Address translation |
| **mangle** | All | Packet modification (TOS, TTL) |
| **raw** | PREROUTING, OUTPUT | Connection tracking bypass |

### 🏗️ Architecture Diagram

```
PACKET FLOW THROUGH NETFILTER:

  Incoming packet
       │
       ▼
  ┌───────────────────┐
  │  PREROUTING       │ ← nat table: DNAT (port forwarding)
  │  (raw, mangle,nat)│
  └────────┬──────────┘
           │
    ┌──────┴──────┐
    │ Routing     │
    │ Decision    │
    └──┬─────┬───┘
       │     │
   LOCAL    FORWARD
    HOST     │
       │     ▼
       │  ┌──────────────────┐
       │  │  FORWARD chain   │ ← filter table: allow/drop forwarded packets
       │  │  (filter table)  │   (e.g., Docker containers → internet)
       │  └────────┬─────────┘
       │           │
       ▼           ▼
  ┌─────────┐  ┌──────────────────┐
  │  INPUT  │  │  POSTROUTING     │ ← nat table: SNAT/MASQUERADE
  │  chain  │  │  (mangle,nat)    │
  │(filter) │  └──────────────────┘
  └────┬────┘
       │
  Local process (app)
       │
       ▼
  ┌─────────┐
  │  OUTPUT │
  │  chain  │
  └─────────┘
```

### ⚙️ Key Commands

```bash
# ══════════════════════════════════════════
# ── iptables ──────────────────────────────
# ══════════════════════════════════════════

# View rules
iptables -L                      # list filter table (INPUT/OUTPUT/FORWARD)
iptables -L -n -v --line-numbers  # numeric + verbose + line numbers
iptables -L INPUT -n -v          # just INPUT chain
iptables -t nat -L -n -v         # nat table

# Add rules (-A = append to end, -I = insert at top/line number)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # allow SSH in
iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # allow HTTP in
iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # allow HTTPS in
iptables -A INPUT -i lo -j ACCEPT                # allow loopback
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT  # allow established
iptables -A INPUT -j DROP                        # drop everything else (default deny)

# Insert at position (before DROP rule):
iptables -I INPUT 5 -p tcp --dport 8080 -j ACCEPT  # insert at line 5

# Delete rules
iptables -D INPUT -p tcp --dport 8080 -j ACCEPT   # delete by rule spec
iptables -D INPUT 5                                 # delete line 5

# Flush (clear all rules — USE WITH EXTREME CARE, can lock you out!)
iptables -F           # flush all chains in filter table
iptables -F INPUT     # flush only INPUT chain

# NAT — MASQUERADE (for NAT/routing, e.g., shared internet via one IP)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
echo 1 > /proc/sys/net/ipv4/ip_forward   # must also enable IP forwarding

# DNAT — Port forwarding (forward port 8080 to internal 192.168.1.10:80)
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80

# Save and restore rules (persistent across reboots)
iptables-save > /etc/iptables/rules.v4        # save
iptables-restore < /etc/iptables/rules.v4     # restore
apt install iptables-persistent               # auto-restore on boot (Debian)

# ══════════════════════════════════════════
# ── firewalld (RHEL/CentOS) ───────────────
# ══════════════════════════════════════════
systemctl start firewalld
systemctl enable firewalld

# Zone management (default zone is "public")
firewall-cmd --get-default-zone
firewall-cmd --list-all                       # show all rules in default zone
firewall-cmd --list-all --zone=public         # specific zone

# Add rules (--permanent makes it survive reload; always --reload after)
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --remove-service=telnet
firewall-cmd --reload                         # apply permanent rules

# Port forwarding (8080 → 80)
firewall-cmd --permanent --add-forward-port=port=8080:proto=tcp:toport=80
firewall-cmd --reload

# Rich rules (complex conditions)
# Allow SSH only from specific subnet
firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address="10.0.0.0/8" service name="ssh" accept'
firewall-cmd --reload

# ══════════════════════════════════════════
# ── ufw (Ubuntu) ──────────────────────────
# ══════════════════════════════════════════
ufw enable                      # enable (careful: may block SSH if not allowed first!)
ufw status verbose              # show rules
ufw allow 22/tcp                # allow SSH
ufw allow 80/tcp                # allow HTTP
ufw allow from 10.0.0.0/8 to any port 5432  # PostgreSQL from private network only
ufw deny 23                     # deny telnet
ufw delete allow 80/tcp         # remove rule
ufw reset                       # reset to defaults (disables UFW)
ufw logging on                  # enable logging
```

### 🏭 Real-Time Production Use Case

**Scenario**: Lock down a new production application server — only allow specific traffic.

```bash
#!/bin/bash
# Production server firewall setup — whitelist approach
# CRITICAL: Run this in a script, not interactively — mistakes lock you out

# Save current rules (in case we need to rollback)
iptables-save > /tmp/iptables_backup_$(date +%Y%m%d).txt

# Flush existing
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# Default policies: DROP everything
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT    # usually allow all outbound

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow established/related connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH from management network only
iptables -A INPUT -p tcp --dport 22 -s 10.10.0.0/16 -j ACCEPT

# Allow application traffic
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Allow monitoring (Prometheus node exporter)
iptables -A INPUT -p tcp --dport 9100 -s 10.10.0.0/16 -j ACCEPT

# Log and drop everything else
iptables -A INPUT -j LOG --log-prefix "IPTABLES-DROP: " --log-level 4
iptables -A INPUT -j DROP

# Persist
iptables-save > /etc/iptables/rules.v4
```

### ❌ Common Mistakes

1. **Setting default `DROP` policy before adding ACCEPT rules** — `iptables -P INPUT DROP` immediately drops all traffic including your SSH session. Always add your ACCEPT rules FIRST, then set the default policy, or use `-A` (append) to add a final DROP rule instead of changing the default policy.
2. **Forgetting `firewall-cmd --reload`** — `--permanent` rules aren't active until you `--reload`. The running config and the permanent config are separate in firewalld.
3. **`iptables -F` without understanding consequences** — Flushing all rules removes all protection AND all connection-tracking. On a live server, this can drop all established connections and expose services. Always back up with `iptables-save` first.
4. **Not accounting for Docker's iptables rules** — Docker modifies iptables extensively. Running `iptables -F` on a Docker host breaks all container networking. Docker adds rules to the `DOCKER` chain and FORWARD chain that ufw/firewalld don't manage.

### 🎤 Interview Questions & Answers

**Q: Explain the difference between INPUT, OUTPUT, and FORWARD chains in iptables.**
A: These are the three built-in chains in the **filter** table, representing different traffic flows. **INPUT**: applies to packets destined for the local system itself (traffic TO this server — incoming connections, pings, etc.). **OUTPUT**: applies to packets originating from the local system (traffic FROM this server — outbound web requests, DNS queries, etc.). **FORWARD**: applies to packets passing THROUGH this system (not destined for nor originating from this system — relevant only when the machine acts as a router, VPN gateway, or firewall between networks). Docker heavily uses the FORWARD chain to route traffic between containers and the host network.

**Q: How do iptables and Docker interact? What should a DevOps engineer know?**
A: Docker directly manipulates iptables to implement container networking. When Docker starts, it creates a `DOCKER` chain and adds rules to `FILTER FORWARD` and `NAT PREROUTING`/`POSTROUTING`. This means: (1) `ufw` or `firewalld` rules may NOT protect containers — Docker's rules in PREROUTING/DOCKER chain bypass them; (2) `iptables -F` on a Docker host kills all container networking; (3) publishing a port (`-p 8080:80`) creates a DNAT PREROUTING rule making it accessible on ALL interfaces, even if `ufw` blocks that port. To properly firewall a Docker host: use Docker's `--iptables=false` and manage rules manually, or configure the `DOCKER-USER` chain (rules there run before Docker's rules), or use network namespaces.

---

## 24. SSH — Deep Dive

### 📖 Concept Explanation

**SSH (Secure Shell)** is the standard protocol for remote server administration. It provides encrypted communication, authentication, and tunneling. Mastery of SSH is non-negotiable for production DevOps work.

**Authentication methods** (in preference order for security):
1. **Public key**: Cryptographic key pair — most secure, recommended
2. **Certificate**: SSH CA-signed certificates — scalable for large environments  
3. **Password**: Password-based — should be disabled in production
4. **Host-based**: Trust based on source host — avoid in most cases

### 🏗️ Architecture Diagram

```
SSH KEY AUTHENTICATION FLOW:

  CLIENT                              SERVER
  ──────                              ──────
  ~/.ssh/id_ed25519     ←──────────── ~/.ssh/authorized_keys
  (private key)         public key    (contains client's public key)
        │               stored here
        │
  SSH handshake:
  1. Client → Server: "I have a key"
  2. Server → Client: "Prove it — sign this random challenge"
  3. Client signs challenge with private key
  4. Server verifies signature with stored public key
  5. If valid: authenticated

SSH TUNNEL TYPES:
  ┌─────────┐                              ┌─────────────────┐
  │  LOCAL  │  Local port forwarding (-L)  │  REMOTE SERVER  │
  │MACHINE  │ ssh -L 8080:db:5432 jump     │   (jump host)   │
  │         │ ───────────────────────────▶ │        │        │
  │localhost│                              │        ▼        │
  │ :8080   │ ◀─────────────────────────── │    db:5432      │
  └─────────┘  Traffic forwarded locally  └─────────────────┘

  ┌─────────┐  Remote forwarding (-R)      ┌─────────────────┐
  │  LOCAL  │                              │  REMOTE SERVER  │
  │MACHINE  │◀──────────────────────────── │  remote:9000    │
  │  :3000  │ Port on remote forwards here │                  │
  └─────────┘                              └─────────────────┘

  BASTION HOST ARCHITECTURE:
  ┌──────────┐   SSH    ┌─────────────┐   SSH    ┌────────────────┐
  │  Laptop  │─────────▶│   Bastion   │─────────▶│ Private Server │
  │(internet)│          │  (jump host)│          │  (10.0.0.x)    │
  └──────────┘          │  (public IP)│          └────────────────┘
                        └─────────────┘
  Single hardened entry point — all access through bastion
```

### ⚙️ Key Commands

```bash
# ─── Key generation ───────────────────────────────────────────
# Ed25519: modern, fast, small key — PREFERRED
ssh-keygen -t ed25519 -C "alice@company.com" -f ~/.ssh/id_ed25519

# RSA 4096: wide compatibility (older systems)
ssh-keygen -t rsa -b 4096 -C "alice@company.com" -f ~/.ssh/id_rsa

# ECDSA: elliptic curve (good balance)
ssh-keygen -t ecdsa -b 521 -f ~/.ssh/id_ecdsa

# Deploy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
# Or manually:
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# ─── SSH client configuration (~/.ssh/config) ────────────────
cat << 'EOF' > ~/.ssh/config
# Global defaults
Host *
    ServerAliveInterval 60     # keep-alive every 60s
    ServerAliveCountMax 3      # disconnect after 3 missed keep-alives
    AddKeysToAgent yes         # add key to ssh-agent automatically
    IdentityFile ~/.ssh/id_ed25519

# Bastion host
Host bastion
    HostName 203.0.113.10
    User ec2-user
    IdentityFile ~/.ssh/aws_prod_key

# Jump through bastion to private servers
Host *.prod.internal
    User ubuntu
    ProxyJump bastion          # connect via bastion host
    IdentityFile ~/.ssh/aws_prod_key

# Specific host with custom port
Host legacy-db
    HostName 192.168.1.50
    User dba
    Port 2222
    IdentityFile ~/.ssh/legacy_key
EOF
chmod 600 ~/.ssh/config

# Usage after config:
ssh bastion                   # direct to bastion
ssh web01.prod.internal       # jumps through bastion automatically
ssh legacy-db                 # uses all custom settings

# ─── ProxyJump (bastion host) ─────────────────────────────────
# One-liner (no config file needed)
ssh -J bastion user@private-server
# Multiple hops
ssh -J bastion1,bastion2 user@final-server

# ─── Port Forwarding ──────────────────────────────────────────
# Local forwarding (-L): access remote service locally
# Access remote PostgreSQL as if it were local:
ssh -L 15432:db.prod.internal:5432 bastion
# Now: psql -h localhost -p 15432 -U myuser mydb

# Access remote web app locally (during maintenance):
ssh -L 8080:localhost:80 webserver

# Remote forwarding (-R): expose local service on remote server
# Expose local dev server (port 3000) on remote server's port 9000
ssh -R 9000:localhost:3000 remote-server

# Dynamic forwarding (-D): SOCKS proxy
ssh -D 1080 bastion    # use bastion as SOCKS5 proxy (all traffic)
# Configure browser proxy: SOCKS5 127.0.0.1:1080
# Now browse as if from bastion

# Keep tunnels alive:
ssh -fN -L 15432:db:5432 bastion   # -f background, -N no command

# ─── SSH agent ────────────────────────────────────────────────
eval "$(ssh-agent -s)"           # start agent
ssh-add ~/.ssh/id_ed25519        # add key to agent
ssh-add -l                       # list keys in agent

# Agent forwarding (use your local keys on remote servers — USE WITH CAUTION)
ssh -A bastion           # forward agent to bastion
# From bastion, you can SSH to internal servers using your local keys
# Security risk: root on bastion can hijack your agent socket!
# Prefer ProxyJump over agent forwarding.

# ─── SSHD hardening (/etc/ssh/sshd_config) ────────────────────
grep -v "^#\|^$" /etc/ssh/sshd_config  # show non-comment lines

# Production hardened sshd_config settings:
cat << 'EOF'
Port 22                          # consider non-standard (security by obscurity, minor)
AddressFamily inet               # IPv4 only (if not using IPv6)
PermitRootLogin no               # NEVER allow direct root login
PasswordAuthentication no        # KEY AUTH ONLY — disable passwords
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3                   # 3 wrong attempts = disconnect
MaxSessions 10
LoginGraceTime 20                # 20 seconds to authenticate
X11Forwarding no                 # disable (attack vector)
AllowTcpForwarding yes           # needed for tunnels (restrict if not needed)
AllowAgentForwarding no          # disable agent forwarding (security)
ClientAliveInterval 300          # keepalive every 5 min
ClientAliveCountMax 2            # disconnect after 2 missed keepalives
UseDNS no                        # don't do reverse DNS (faster logins)
AllowUsers deploy alice bob      # whitelist specific users
AllowGroups sshaccess            # or whitelist by group
Banner /etc/ssh/banner           # display legal/warning banner before login
EOF

# Apply changes (test config first!)
sshd -t                          # test configuration syntax
systemctl reload sshd            # reload without dropping connections

# ─── Brute force protection with fail2ban ─────────────────────
apt install fail2ban
cat << 'EOF' > /etc/fail2ban/jail.d/sshd.conf
[sshd]
enabled = true
port = ssh
maxretry = 5
bantime = 3600    # ban for 1 hour
findtime = 600    # 5 failures within 10 minutes
EOF
systemctl enable --now fail2ban
fail2ban-client status sshd      # check banned IPs
fail2ban-client set sshd unbanip 1.2.3.4  # unban specific IP
```

### 🏭 Real-Time Production Use Case

**Scenario**: Large AWS deployment — 500 EC2 instances, 50 engineers needing access. Managing SSH keys individually is unscalable.

**Solution: SSH Certificate Authority (CA)**

```bash
# Create SSH CA (done ONCE, keep private key VERY secure)
ssh-keygen -t ed25519 -f /secure/ssh_ca -C "MyCompany SSH CA"

# Add CA public key to all servers (deploy via cloud-init or Ansible):
echo "TrustedUserCAKeys /etc/ssh/ssh_ca.pub" >> /etc/ssh/sshd_config
cp ssh_ca.pub /etc/ssh/ssh_ca.pub

# Sign an engineer's public key (valid for 1 day, specific username)
ssh-keygen -s /secure/ssh_ca \
  -I "alice@mycompany" \        # key identifier (for audit)
  -n alice,ubuntu \              # valid usernames on the server
  -V +1d \                       # valid for 1 day
  alice_id_ed25519.pub           # signs alice's public key

# Alice now has: alice_id_ed25519-cert.pub (the certificate)
# She doesn't need to be in any authorized_keys file on any server
# Certificate expiry enforces time-limited access automatically
```

**Result**: Revocation is instant (remove CA key from servers), access is time-limited, and audit trail is excellent (certificate includes identity string).

### ❌ Common Mistakes

1. **Agent forwarding to untrusted hosts** — `ssh -A` forwards your SSH agent socket to the remote server. Any user with root access on that server can connect to your socket and use your keys to authenticate anywhere. Prefer `ProxyJump` which doesn't forward the agent.
2. **`chmod` on SSH files wrong** — SSH is very strict about permissions. `~/.ssh` must be `700`, `~/.ssh/authorized_keys` must be `600`, `~/.ssh/config` must be `600`. Wrong permissions = SSH silently ignores the file.
3. **Not testing `sshd_config` before reloading** — A syntax error in sshd_config can prevent new SSH connections. If you're working remotely, keep a second SSH session open while testing changes. Always run `sshd -t` before `systemctl reload sshd`.
4. **`PermitRootLogin yes`** — Root login should ALWAYS be disabled in production. Use a regular user and `sudo`. If you need root on a newly built server, use `PermitRootLogin without-password` only during initial setup, then disable it.
5. **Not rotating SSH keys** — SSH keys should be rotated periodically (annually at minimum) or immediately when a team member leaves. SSH CA certificates make this much easier (revoke the certificate, issue new one).

### 🎤 Interview Questions & Answers

**Q: Explain SSH port forwarding. Give a production use case for local forwarding.**
A: SSH **local forwarding** (`-L local_port:remote_host:remote_port`) creates a tunnel where traffic sent to `local_port` on your machine is forwarded through the SSH connection to `remote_host:remote_port` (as seen from the SSH server). Production use case: accessing a database in a private subnet with no public endpoint. `ssh -L 5432:db.prod.internal:5432 bastion-host` — now `psql -h localhost -p 5432` connects through the bastion to the private database. This avoids opening the database port to the internet while allowing controlled developer access. It's also used for accessing admin UIs (Elasticsearch, Kibana, Prometheus) that are intentionally not exposed publicly.

**Q: What is a bastion host and why do you use one?**
A: A **bastion host** (jump server) is a hardened, dedicated server with a public IP that acts as the single entry point for SSH access to a private network. All private servers only accept SSH connections FROM the bastion. Benefits: (1) Reduced attack surface — only one host exposed to internet for SSH; (2) Centralized logging — all SSH access goes through one point, audit trail in one place; (3) IP whitelisting — only bastion's IP in security groups/firewall rules for all private servers; (4) MFA enforcement — implement MFA on bastion only; (5) Key management — only bastion's public key needed in all servers' `authorized_keys`. Modern implementation uses `ProxyJump` for transparent access: `ssh -J bastion user@private-server` — the client SSHs to the bastion then immediately SSHs to the final destination, appearing seamless to the user.

**Q: Your private key is accidentally exposed. What do you do immediately?**
A: Immediate steps: (1) **Identify all places the public key is installed** — `grep -r "$(ssh-keygen -y -f id_ed25519 | cut -d' ' -f2)" /home/*/`.ssh/authorized_keys on all servers, plus cloud provider key pairs. (2) **Remove the key from all authorized_keys files immediately** — on every server using the exposed key. (3) **Check audit logs** for any unauthorized access using that key: `grep -r "Accepted publickey" /var/log/auth.log`. (4) **Generate new key pair** and deploy the new public key. (5) **Rotate any secrets accessible from the compromised key** — assume the attacker may have had access. (6) **Review/audit** what the key had access to. (7) If using SSH CA certificates: revoke the certificate and issue a new one. This is why SSH CA is better than individual key distribution at scale.

---

## 25. DNS & Name Resolution

### 📖 Concept Explanation

**DNS (Domain Name System)** translates human-readable hostnames to IP addresses. Linux has multiple layers of name resolution, configured by `/etc/nsswitch.conf`.

**DNS Record Types**:
| Record | Purpose | Example |
|---|---|---|
| **A** | IPv4 address | `app.example.com → 1.2.3.4` |
| **AAAA** | IPv6 address | `app.example.com → 2001:db8::1` |
| **CNAME** | Canonical name (alias) | `www → app.example.com` |
| **MX** | Mail exchange | `example.com → mail.example.com` |
| **TXT** | Text records (SPF, DKIM, verification) | `v=spf1 include:...` |
| **NS** | Nameserver | `example.com nameserver → ns1.example.com` |
| **PTR** | Reverse DNS (IP → hostname) | `1.2.3.4 → app.example.com` |
| **SRV** | Service locator | `_http._tcp → host + port` |
| **SOA** | Start of Authority | Zone metadata |

### 🏗️ Architecture Diagram

```
DNS RESOLUTION FLOW:

  Application: "What is the IP of api.mycompany.com?"
       │
       ▼
  ┌───────────────────────────────────────────────────┐
  │  Check /etc/nsswitch.conf: "hosts: files dns"     │
  └────────────────────┬──────────────────────────────┘
                       │
       ┌───────────────┴───────────────┐
       │                               │
  Check /etc/hosts             Not found in /etc/hosts
  (instant, static)                    │
       │ "api.mycompany.com not here"  │
       └───────────────┬───────────────┘
                       │
       ┌───────────────▼──────────────────────────────┐
       │  Query nameserver from /etc/resolv.conf       │
       │  nameserver 10.0.0.2 (internal corporate DNS) │
       └────────────────────────────┬─────────────────┘
                                    │
       ┌────────────────────────────▼─────────────────┐
       │  DNS Resolution Chain:                        │
       │  1. Check local cache (systemd-resolved)      │
       │  2. Query Authoritative NS → api.mycompany.com│
       │  3. Cache result (TTL seconds)                │
       └──────────────────────────────────────────────┘

TTL (Time-To-Live):
  Short TTL (60-300s)  → fast failover, more DNS queries
  Long TTL (3600s+)    → efficient, slow failover
  BEFORE MIGRATIONS: lower TTL to 60s 24 hours before change!
```

### ⚙️ Key Commands

```bash
# ─── dig (DNS Information Groper) — primary tool ─────────────
dig google.com                      # query A record (default)
dig google.com A                    # explicit A record query
dig google.com MX                   # MX records
dig google.com TXT                  # TXT records (SPF, DKIM)
dig google.com NS                   # nameservers
dig google.com ANY                  # all records (many servers block this)
dig -x 8.8.8.8                      # reverse DNS lookup (PTR record)

# Query specific nameserver (bypass system resolver)
dig @8.8.8.8 google.com             # query Google's DNS directly
dig @10.0.0.2 app.internal          # query internal corporate DNS
dig @ns1.google.com google.com      # query authoritative nameserver

# Trace full DNS resolution chain
dig +trace google.com               # shows root → TLD → authoritative path

# Short answers
dig +short google.com               # just the IP(s)
dig +short google.com MX            # just MX records
dig +noall +answer google.com       # only the answer section

# Show TTL
dig google.com | grep -A2 "ANSWER SECTION"
# ;; ANSWER SECTION:
# google.com.    299    IN    A    142.250.80.14
#                 ^^^   TTL

# Batch query
dig -f /tmp/hosts_to_query.txt +short

# ─── nslookup (legacy but still widely used) ──────────────────
nslookup google.com                 # basic lookup
nslookup google.com 8.8.8.8        # using specific nameserver
nslookup -type=MX google.com       # specific record type
nslookup 8.8.8.8                   # reverse lookup

# ─── host ─────────────────────────────────────────────────────
host google.com                     # brief A/AAAA/MX lookup
host -t MX google.com              # MX records
host 8.8.8.8                       # reverse lookup

# ─── systemd-resolved ─────────────────────────────────────────
resolvectl status                   # DNS config per interface
resolvectl query google.com         # query (uses systemd-resolved)
resolvectl statistics               # cache hit/miss stats
resolvectl flush-caches             # flush DNS cache
resolvectl log-level debug          # enable debug logging (temporary)

# Clear DNS cache (distribution-specific):
systemctl restart systemd-resolved  # Ubuntu/systemd
rndc flush                          # BIND DNS server
service nscd restart                # nscd cache daemon

# ─── /etc/hosts — for critical overrides ─────────────────────
# Override DNS (checked first per nsswitch.conf "files" entry):
cat /etc/hosts
# 127.0.0.1  localhost
# 10.0.1.50  db.prod.internal db  # internal service
# 10.0.1.60  api.prod.internal api

# ─── dnsmasq (lightweight local DNS) ─────────────────────────
apt install dnsmasq
cat << 'EOF' > /etc/dnsmasq.conf
# Forward to upstream DNS
server=8.8.8.8
server=8.8.4.4

# Cache 1000 entries
cache-size=1000

# Local domain resolution
address=/dev.local/127.0.0.1       # wildcard: *.dev.local → 127.0.0.1
EOF
systemctl restart dnsmasq
```

### 🏭 Real-Time Production Use Case

**Scenario**: During a cloud migration, DNS changes need to propagate quickly to minimize downtime.

```bash
# BEFORE migration (24 hours before):
# 1. Check current TTL
dig +short myapp.example.com
dig myapp.example.com | grep -i ttl   # likely 3600 (1 hour)

# 2. Lower TTL to 60 seconds in DNS provider
# (change TTL value from 3600 to 60 in DNS records)
# Wait one full TTL period (1 hour) for old TTL to expire everywhere

# DURING migration:
# 3. Change DNS record to new IP
# 4. Verify from multiple locations:
dig @8.8.8.8 myapp.example.com +short   # Google DNS
dig @1.1.1.1 myapp.example.com +short   # Cloudflare DNS
dig @208.67.222.222 myapp.example.com +short  # OpenDNS

# 5. Monitor for old IP still appearing (stale caches)
for i in $(seq 1 10); do
    echo "$(date): $(dig +short @8.8.8.8 myapp.example.com)"
    sleep 10
done

# AFTER migration:
# 6. Raise TTL back to 3600
```

### 🔧 Troubleshooting Tips

```
Problem: DNS resolution failing for internal hostnames
Symptom: dig internal.company.com SERVFAIL or NXDOMAIN
Fix:
  cat /etc/resolv.conf              # check nameservers and search domains
  dig @<internal_dns_ip> internal.company.com  # test directly against internal DNS
  resolvectl status                 # check which DNS server interface is using
  # Often: split-DNS configuration needed:
  # Internal DNS for .company.com, public DNS for everything else
  # Fix via NetworkManager: add company.com to "DNS Search" domains with internal DNS
```

```
Problem: DNS caching causing stale records after update
Symptom: New IP is in DNS but some servers still get old IP
Fix:
  resolvectl flush-caches          # flush systemd-resolved cache
  resolvectl statistics            # check cache entries
  # Check TTL:
  dig myapp.example.com | grep -i ttl
  # If TTL=3600, you may have to wait up to 1 hour for full propagation
  # Short-term override while waiting:
  echo "NEW_IP myapp.example.com" >> /etc/hosts  # /etc/hosts takes priority
```

### 🎤 Interview Questions & Answers

**Q: What is the order of DNS resolution in Linux, and how do you configure it?**
A: The order is controlled by `/etc/nsswitch.conf`, specifically the `hosts:` line, which typically reads `hosts: files dns` or `hosts: files mdns4_minimal [NOTFOUND=return] dns`. (1) **files** — checks `/etc/hosts` first. Entries here always take precedence over DNS. (2) **dns** — queries the nameservers in `/etc/resolv.conf`. (3) **mdns** — mDNS (Avahi/Bonjour) for .local hostnames. On modern Ubuntu with systemd-resolved, the stub resolver listens on `127.0.0.53` and handles caching and split-DNS. You can override nameservers per-interface with `nmcli` or `resolvectl`.

**Q: How do you diagnose a DNS issue in production? Walk me through your steps.**
A: Step 1: Check if the issue is DNS or something else — `ping -c1 IP_ADDRESS` (bypasses DNS). If that works, the issue is DNS. Step 2: `dig hostname` — check the answer and error code. NXDOMAIN = name doesn't exist. SERVFAIL = server error. No response = connectivity to DNS server. Step 3: `dig @8.8.8.8 hostname` — query public DNS to check if the record exists globally vs. internally. Step 4: `cat /etc/resolv.conf` — verify correct nameservers are configured. Step 5: `resolvectl statistics` — check if systemd-resolved has cache issues. Step 6: `dig +trace hostname` — traces the full resolution chain from root servers to identify where it fails. Step 7: For internal DNS, `dig @internal_dns_server hostname` to test the internal DNS directly.

---

# 🔥 Sections 4–6 Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────┐
│           LINUX SECTIONS 4–6 — QUICK REFERENCE                      │
├────────────────────────────┬────────────────────────────────────────┤
│ TOPIC                      │ KEY COMMAND / FACT                     │
├────────────────────────────┼────────────────────────────────────────┤
│ Kill gracefully first      │ kill -TERM pid → wait → kill -9 pid   │
│ Find D-state processes     │ ps aux | grep " D "                    │
│ Find zombie processes      │ ps aux | awk '$8~/Z/{print}'           │
│ Reload nginx config        │ kill -HUP $(cat /var/run/nginx.pid)   │
│ What file is open on port  │ lsof -i :8080                         │
│ Find deleted-but-open files│ lsof | grep deleted                   │
│ Trace syscalls             │ strace -p PID                         │
│ Custom systemd service     │ /etc/systemd/system/app.service        │
│ After editing unit file    │ systemctl daemon-reload                 │
│ Check service journal      │ journalctl -u service -f               │
│ List all timers            │ systemctl list-timers                  │
│ Cron debug                 │ grep CRON /var/log/syslog              │
│ Install + hold version     │ apt install nginx=1.24 → apt-mark hold│
│ What package owns file     │ dpkg -S /path OR rpm -qf /path        │
│ Package dependencies       │ apt-cache rdepends nginx               │
│ Network interface info     │ ip addr show / ip -s link show eth0   │
│ Routing decision           │ ip route get 8.8.8.8                  │
│ Who's on port 80           │ ss -tlnp | grep :80                   │
│ Test port connectivity     │ nc -zv host 443                       │
│ Capture packets            │ tcpdump -i eth0 port 80 -w file.pcap  │
│ Find packet loss           │ mtr --report 8.8.8.8                  │
│ iptables list rules        │ iptables -L -n -v --line-numbers       │
│ firewalld open port        │ firewall-cmd --permanent --add-port=80/tcp && --reload │
│ SSH key gen                │ ssh-keygen -t ed25519 -C "user@host"  │
│ SSH through bastion        │ ssh -J bastion user@private           │
│ Local port forward         │ ssh -L 5432:db:5432 bastion           │
│ Test sshd config           │ sshd -t                               │
│ DNS lookup                 │ dig @8.8.8.8 host.com +short          │
│ Flush DNS cache            │ resolvectl flush-caches               │
│ Trace DNS resolution       │ dig +trace host.com                   │
│ Check TTL                  │ dig host.com | grep -i ttl            │
└────────────────────────────┴────────────────────────────────────────┘
```

---

*End of Sections 4–6 | Next: Sections 7–9 cover Storage & Filesystem, Performance & Tuning, and Security Hardening*
