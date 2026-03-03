# 🐧 Linux for DevOps Engineers — Interview-Ready Study Notes
## Sections 1–3: Fundamentals, Shell & Scripting, File Permissions & User Management

> Written from the perspective of a Senior DevOps Architect with 15+ years of enterprise production experience.
> Target audience: Mid-level engineers preparing for Senior DevOps / Linux Engineer interviews.

---

# 🟢 SECTION 1 — LINUX FUNDAMENTALS

---

## 1. Linux Introduction & Distributions

### 📖 Concept Explanation

**Linux** is an open-source, Unix-like operating system **kernel** originally written by Linus Torvalds in 1991. It's important to distinguish three layers:

- **Kernel**: The core program managing hardware (CPU, memory, I/O, processes). File: `/boot/vmlinuz-*`
- **Operating System**: Kernel + GNU tools + init system + libraries (glibc, etc.)
- **Distribution (Distro)**: A packaged OS — kernel + package manager + default config + support model (e.g., Ubuntu, RHEL)

Think of it this way:
- Kernel = Engine
- OS = Engine + chassis + transmission
- Distro = Fully assembled, branded car ready to drive

### 🎯 Why It Is Used (Real-World Context)

Linux powers **~96% of the world's top 1 million web servers**, **100% of the top 500 supercomputers**, and is the foundation of every major container runtime. DevOps engineers use it because:
- Free and open-source (no per-seat licensing like Windows Server)
- Stable and predictable in production
- Scriptable and automatable at every layer
- Native home of containers (namespaces and cgroups are Linux kernel features)

### 🏗️ Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     LINUX DISTRIBUTION                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              USER SPACE                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │ Bash/ZSH │  │  apt/dnf │  │ systemd / init   │  │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘  │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │          GNU C Library (glibc)               │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │  System Call Interface (syscalls)  │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │              KERNEL SPACE                            │   │
│  │  ┌────────────┐ ┌──────────┐ ┌────────────────────┐ │   │
│  │  │Process Mgr │ │Mem Mgmt  │ │  VFS / Filesystems │ │   │
│  │  └────────────┘ └──────────┘ └────────────────────┘ │   │
│  │  ┌────────────┐ ┌──────────────────────────────────┐ │   │
│  │  │ Network    │ │     Device Drivers               │ │   │
│  │  └────────────┘ └──────────────────────────────────┘ │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                    │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │              HARDWARE                                │   │
│  │    CPU        RAM        Disk       NIC       USB    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### ⚙️ Key Commands

```bash
# Check kernel version
uname -r              # e.g., 6.5.0-15-generic

# Full system info
uname -a              # kernel name, hostname, kernel release, machine type

# Check distro name and version
cat /etc/os-release   # standard across all modern distros
lsb_release -a        # Debian/Ubuntu specific but often available

# Check glibc version (critical for binary compatibility)
ldd --version

# Check running kernel modules
lsmod | head -20
```

### 🏭 Distro Comparison Table

| Distro | Package Mgr | Init System | Release Model | Best For |
|---|---|---|---|---|
| **Ubuntu 22.04 LTS** | apt (deb) | systemd | LTS (5yr support) | Dev workstations, GCP/AWS |
| **RHEL 9 / CentOS Stream 9** | dnf (rpm) | systemd | Enterprise stable | Banks, telcos, regulated industries |
| **Debian 12** | apt (deb) | systemd | Stable + Backports | Servers needing rock-solid stability |
| **Alpine 3.x** | apk | OpenRC | Rolling | Docker base images (5MB!) |
| **Amazon Linux 2023** | dnf (rpm) | systemd | Amazon-managed | AWS EC2, ECS optimised |
| **Fedora 40** | dnf (rpm) | systemd | Rapid (6mo) | Cutting-edge features, testing |

> **Production tip**: RHEL/CentOS Stream dominates in finance/telco due to SELinux maturity and long enterprise support. Ubuntu dominates in cloud-native/startup environments. Alpine is the go-to for container base images due to its minimal attack surface.

### 🏭 Real-Time Production Use Case

**Scenario**: A fintech company migrating 200 VMs from CentOS 7 (EOL December 2024) to a supported platform.

- **Problem**: CentOS 7 reached end-of-life. No security patches, compliance failure (PCI-DSS requires supported OS).
- **Solution**: Evaluated RHEL 9 (licensing cost), Rocky Linux 9 (free RHEL clone), and Ubuntu 22.04 LTS.
- **Decision**: Chose Rocky Linux 9 for application servers (rpm ecosystem parity with existing ansible roles) and Ubuntu 22.04 LTS for new container-based microservices.
- **Result**: Zero compliance findings in next audit. Migration completed in 8 weeks using Ansible playbooks.

### ❌ Common Mistakes

1. **Confusing kernel version with distro version** — `uname -r` shows the kernel, not the distro. Ubuntu 22.04 may run kernel 5.15 or 6.x depending on HWE updates. Always check both.
2. **Using CentOS 8 in production** — CentOS 8 EOL was December 2021. Many engineers still confuse CentOS Stream (rolling) with the old stable CentOS. Use Rocky/AlmaLinux as drop-in replacements.
3. **Running Alpine in production VMs** — Alpine uses musl libc instead of glibc. Some Go/Python apps behave differently. Alpine is excellent for containers but verify compatibility before VM use.
4. **Ignoring LTS vs non-LTS** — Ubuntu 23.10 is non-LTS (9 months support). Use 22.04 or 24.04 LTS for production.

### 🔧 Troubleshooting Tips

```
Problem: "Which distro is this server running?"
Symptom: You SSH into an unfamiliar server
Root Cause: You don't have context
Fix:
  cat /etc/os-release
  hostnamectl          # shows OS + kernel + architecture
  cat /proc/version    # kernel compile info
```

```
Problem: Binary won't run - "No such file or directory" even though file exists
Symptom: ./myapp returns error
Root Cause: Binary compiled for glibc but system uses musl (Alpine) or wrong arch
Fix:
  file ./myapp             # shows ELF type, architecture, dynamic linker path
  ldd ./myapp              # shows required shared libraries
  ls /lib/ld-*             # check which dynamic linker is present
```

### 🎤 Interview Questions & Answers

**Q: What is the difference between a Linux kernel, OS, and distribution?**
A: The **kernel** is the core program that interfaces with hardware — it manages processes, memory, filesystems, and device drivers. The **operating system** is the kernel plus the userland tools (GNU coreutils, shells, libraries like glibc) needed to run programs. A **distribution** packages the OS with a specific init system (systemd), package manager (apt/dnf), default configuration, and support model. For example: Linux kernel 6.5 + GNU tools + systemd + apt = Ubuntu 22.04 LTS.

**Q: Why does Linux dominate servers over Windows?**
A: Several factors: (1) Cost — no per-seat licensing at scale saves millions; (2) Stability — Linux servers routinely achieve 5+ year uptimes; (3) Security model — granular permissions, SELinux/AppArmor, minimal attack surface; (4) Scripting — every configuration is a text file, enabling full IaC automation; (5) Container-native — Docker, Kubernetes, and cgroups are Linux kernel features; (6) Open source — full visibility into OS behaviour, critical for compliance.

**Q: What is the difference between CentOS Stream and Rocky Linux?**
A: **CentOS Stream** is now a rolling-release distribution positioned *upstream* of RHEL — it gets updates before RHEL, meaning it's slightly less stable. It's useful for testing what's coming in the next RHEL release. **Rocky Linux** is a downstream RHEL clone (like the original CentOS was) — it tracks RHEL releases closely, is binary compatible, and is suitable for production where CentOS used to be.

**Q: What is Alpine Linux and why is it popular for Docker?**
A: Alpine Linux is a security-oriented, minimal Linux distribution using musl libc and BusyBox. Its base Docker image is ~5MB vs ~72MB for Ubuntu. Smaller images mean faster pulls, smaller attack surface, and lower registry storage costs. However, it uses musl instead of glibc which can cause subtle compatibility issues with some applications compiled against glibc.

---

## 2. Linux Architecture

### 📖 Concept Explanation

Linux architecture is divided into two fundamental protection domains: **kernel space** and **user space**.

- **Kernel space**: Runs in CPU privilege ring 0. Has direct hardware access. A crash here = kernel panic = system down.
- **User space**: Runs in CPU privilege ring 3. All applications, shells, and most services run here. A crash here = one process dies, system survives.

The bridge between them is the **system call interface (syscall)**. When a userspace program needs hardware access (read a file, send a network packet, allocate memory), it makes a syscall — a controlled entry into kernel space.

### 🎯 Why It Is Used

This separation is fundamental to OS security and stability. A buggy application cannot corrupt kernel memory. This isolation is also how containers work — containers share the kernel but are isolated in user space via namespaces and cgroups.

### 🏗️ Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER SPACE (Ring 3)                           │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐   │
│  │  Web Server  │  │  Database    │  │  Shell (bash/zsh)   │   │
│  │  (nginx)     │  │  (postgres)  │  │  User Applications  │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬──────────┘   │
│         │                 │                       │               │
│  ┌──────▼─────────────────▼───────────────────────▼──────────┐  │
│  │           GNU C Library / glibc (libc.so)                  │  │
│  │    Provides wrappers around raw system calls               │  │
│  └──────────────────────────────┬─────────────────────────────┘  │
└─────────────────────────────────┼───────────────────────────────┘
                                  │  SYSCALL INTERFACE
                    ┌─────────────▼─────────────────┐
                    │   int 0x80 / syscall instr.    │
                    │  (controlled entry to kernel)  │
                    └─────────────┬─────────────────┘
┌─────────────────────────────────┼───────────────────────────────┐
│                    KERNEL SPACE (Ring 0)                          │
│         ┌───────────────────────▼─────────────────────────┐     │
│         │             System Call Handler                   │     │
│         └───┬────────────┬──────────────┬──────────────────┘     │
│             ▼            ▼              ▼                         │
│  ┌─────────────┐  ┌───────────┐  ┌──────────────────────────┐   │
│  │  Process    │  │  Memory   │  │  VFS (Virtual Filesystem) │   │
│  │  Scheduler  │  │  Manager  │  │  ext4 / xfs / tmpfs      │   │
│  └──────┬──────┘  └─────┬─────┘  └─────────────┬────────────┘   │
│         │               │                        │                │
│  ┌──────▼───────────────▼────────────────────────▼─────────┐    │
│  │                    Device Drivers                         │    │
│  │    Block (sda/nvme)    Network (eth0)    USB / PCIe      │    │
│  └──────────────────────────────┬────────────────────────────┘   │
└─────────────────────────────────┼────────────────────────────────┘
                                  ▼
              ┌─────────────────────────────────────┐
              │             HARDWARE                 │
              │   CPU    RAM    NVMe    NIC    GPU   │
              └─────────────────────────────────────┘
```

### ⚙️ Key Commands

```bash
# View system calls made by a process (strace)
strace ls /tmp              # trace all syscalls made by `ls`
strace -p 1234              # attach to running process PID 1234
strace -e trace=read,write ls  # filter to specific syscalls

# View kernel messages (kernel space events)
dmesg | tail -50            # last 50 kernel ring buffer messages
dmesg -T | grep -i error    # kernel errors with human timestamps

# Check kernel parameters (kernel space tuning)
sysctl -a | grep net.core   # all network core kernel params
cat /proc/sys/kernel/pid_max  # max PID value (default 32768, can increase)

# Inspect /proc — virtual filesystem exposing kernel data structures
cat /proc/cpuinfo           # CPU info from kernel
cat /proc/meminfo           # Memory stats from kernel
cat /proc/1/status          # Status of PID 1 (systemd)
ls /proc/1/fd               # File descriptors open by PID 1
```

### 🏭 Real-Time Production Use Case

**Scenario**: E-commerce platform, 50-engineer team, experiencing intermittent application crashes.

- **Problem**: A Go microservice was crashing with "segmentation fault" but the application code looked clean.
- **Investigation**: Used `strace -p <pid>` and saw `mmap()` syscalls returning `ENOMEM`. Checked `/proc/sys/vm/overcommit_memory` = 0 (default — kernel may refuse memory requests under pressure).
- **Root Cause**: The system had vm.overcommit_memory=0 and was under memory pressure. The kernel refused mmap() calls for the Go runtime's memory allocations.
- **Fix**: Tuned `vm.overcommit_memory=1` (always overcommit) on the batch processing hosts. Set proper container memory limits to prevent OOM.
- **Result**: Zero crashes over 30-day observation period.

### ❌ Common Mistakes

1. **Assuming application crashes = application bugs** — Always check kernel logs (`dmesg`) first. OOM killer, hardware errors, and device driver issues in kernel space can manifest as application crashes.
2. **Direct kernel space programming without understanding protection rings** — Running code at ring 0 (kernel modules, eBPF) without memory safety awareness can cause kernel panics and bring down the entire system.
3. **Confusing `/proc` as a real filesystem** — `/proc` is a **virtual** filesystem. Files like `/proc/meminfo` are generated in real-time by the kernel. You cannot back them up or restore them. Writing to `/proc/sys/` changes live kernel parameters.
4. **Ignoring syscall overhead** — High-frequency applications (HFT, real-time video) making millions of syscalls/second can hit performance walls. This is why modern frameworks use io_uring (batch syscalls) or DPDK (bypass kernel network stack entirely).

### 🔧 Troubleshooting Tips

```
Problem: Application hangs indefinitely
Symptom: Process appears running but unresponsive
Root Cause: Often stuck in D-state (uninterruptible sleep) waiting on I/O syscall
Fix:
  ps aux | grep " D "      # find processes in D state
  cat /proc/<pid>/wchan    # shows which kernel function the process is waiting on
  cat /proc/<pid>/stack    # kernel stack trace (shows exact kernel code path)
  # Usually caused by NFS hang, failing disk, or storage backend issue
```

```
Problem: System sluggish, high system CPU (sy%)
Symptom: `top` shows high %sy (system/kernel CPU time)
Root Cause: Excessive syscall frequency — context switching overhead
Fix:
  perf top                  # show top kernel functions consuming CPU
  strace -c -p <pid>        # count syscalls and time spent
  # If it's many read()/write() calls: investigate io_uring or buffered I/O
```

### 🎤 Interview Questions & Answers

**Q: What is a system call and why is it necessary?**
A: A **system call** is the mechanism by which a userspace program requests a service from the kernel. Since user processes run in an unprivileged CPU mode (ring 3) and cannot directly access hardware or kernel data structures, they must make a controlled transition into kernel space via the syscall interface. Common syscalls include `read()`, `write()`, `open()`, `fork()`, `mmap()`, and `socket()`. The glibc library provides C function wrappers around raw syscalls. When you call `open("/etc/hosts", O_RDONLY)` in C, glibc translates that into a `syscall(SYS_open, ...)` instruction.

**Q: What's the difference between kernel space and user space? Why does this matter for DevOps?**
A: **Kernel space** runs at CPU privilege ring 0 with full hardware access — a crash here panics the entire system. **User space** runs at ring 3 with no direct hardware access — a crash kills only that process. This matters for DevOps because: (1) Kernel modules and eBPF programs require extreme care as bugs can crash servers; (2) Container isolation relies on kernel-level namespaces and cgroups — containers share the kernel, so a kernel exploit breaks all containers on the host; (3) Performance tuning often requires adjusting kernel parameters via `/proc/sys/` and `sysctl`.

**Q: How would you diagnose a system call-level problem in a production application?**
A: I'd use `strace` to trace syscalls. First, I'd attach to the running process: `strace -p <pid> -o /tmp/trace.log`. I'd look for error returns (e.g., `-1 EPERM`, `-1 ENOMEM`). If I suspect high syscall frequency causing performance issues, I'd run `strace -c -p <pid>` to get a syscall count summary. For performance analysis without the overhead of strace, I'd use `perf stat -p <pid>` or `perf record + perf report` to profile at the kernel level. In Kubernetes environments, I'd also check if seccomp profiles are blocking syscalls (shows as `EPERM`).

---

## 3. Linux Boot Process (Deep Dive)

### 📖 Concept Explanation

The Linux boot process is a sequential chain of stages, each handing control to the next. Understanding this is critical for:
- Recovering from boot failures
- Optimising boot time (K8s node scaling speed)
- Understanding systemd dependencies
- Security hardening (GRUB password, Secure Boot)

The stages are: **BIOS/UEFI → GRUB → Kernel → initramfs → systemd → Login**

### 🎯 Why It Is Used

In cloud/DevOps contexts, boot time directly affects:
- **Auto-scaling speed**: Slower boot = slower scale-out response
- **Disaster recovery**: After a crash, how fast does a server come back online?
- **Kubernetes**: Nodes need to boot, register with the control plane, and accept pods in under 60–90 seconds for tight SLAs

### 🏗️ Architecture Diagram

```
 POWER ON
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 1: BIOS / UEFI                                   │
│  • POST (Power-On Self Test) — verify hardware          │
│  • BIOS: reads MBR (sector 0) of boot disk             │
│  • UEFI: reads EFI partition (/boot/efi) — faster      │
│  • Hands off to bootloader                              │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 2: GRUB2 (Grand Unified Bootloader)              │
│  • Loads /boot/grub2/grub.cfg                          │
│  • Presents boot menu (multi-boot, recovery option)     │
│  • Loads kernel image (/boot/vmlinuz-*)                │
│  • Loads initial RAM disk (/boot/initramfs-* or initrd) │
│  • Passes kernel parameters (quiet, ro, root=UUID=...)  │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 3: KERNEL INITIALISATION                         │
│  • Kernel decompresses itself                           │
│  • Detects and initialises hardware (CPU, RAM, buses)   │
│  • Mounts initramfs as temporary root (/)               │
│  • Kernel PID 1 = init process starts from initramfs   │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 4: initramfs (Initial RAM Filesystem)            │
│  • Minimal filesystem loaded entirely in RAM            │
│  • Contains drivers needed to access real disk          │
│  • Runs udev to detect devices                         │
│  • Mounts the real root filesystem (/, /dev/sda1 etc.) │
│  • Executes pivot_root to switch to real filesystem     │
│  • Hands off to systemd on real root                   │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 5: systemd (PID 1)                               │
│  • Reads default.target (e.g., multi-user.target)      │
│  • Starts all required units in dependency order        │
│  • Parallelises startup (faster than old SysV init)     │
│  • systemd-journald starts early for logging            │
│  • Reaches target: all services up                      │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 6: Login                                         │
│  • getty spawns on TTYs (/dev/tty1-6)                  │
│  • SSH daemon accepts remote logins                     │
│  • Display manager starts (if GUI: gdm, lightdm)       │
│  • User logs in, bash/zsh spawns                       │
└─────────────────────────────────────────────────────────┘
```

### ⚙️ Key Commands

```bash
# ─── GRUB2 Management ───────────────────────────────────────
# View GRUB config (auto-generated — do NOT edit directly)
cat /boot/grub2/grub.cfg

# Edit GRUB defaults (this IS the file you edit)
vi /etc/default/grub

# After editing /etc/default/grub, regenerate grub.cfg
update-grub                  # Debian/Ubuntu
grub2-mkconfig -o /boot/grub2/grub.cfg  # RHEL/CentOS

# ─── Boot Analysis ──────────────────────────────────────────
# Show total boot time and slowest services
systemd-analyze               # e.g., "Startup finished in 1.234s (kernel) + 4.567s (userspace)"
systemd-analyze blame         # list all services by startup time (slowest first)
systemd-analyze critical-chain  # show the critical path (longest dependency chain)
systemd-analyze plot > boot.svg  # generate SVG visualisation of boot

# ─── systemd Targets (replacement for runlevels) ────────────
# List all targets
systemctl list-units --type=target

# Get/set default boot target
systemctl get-default          # e.g., multi-user.target
systemctl set-default graphical.target  # switch to GUI boot

# Common targets and their SysV equivalents:
# poweroff.target  = runlevel 0
# rescue.target    = runlevel 1 (single-user mode)
# multi-user.target = runlevel 3 (normal, no GUI)
# graphical.target = runlevel 5 (with GUI)
# reboot.target    = runlevel 6

# Boot into rescue mode (one-time)
systemctl isolate rescue.target   # immediate, from running system

# ─── GRUB Rescue Mode (from boot menu) ──────────────────────
# At GRUB menu: press 'e' to edit boot entry
# Add to kernel line: systemd.unit=rescue.target
# Press Ctrl+X to boot with that parameter

# ─── initramfs Management ───────────────────────────────────
# View contents of initramfs
lsinitrd /boot/initramfs-$(uname -r).img | head -40   # RHEL
unmkinitramfs /boot/initrd.img-$(uname -r) /tmp/initrd-contents  # Debian

# Rebuild initramfs (needed after adding drivers for new hardware)
dracut --force   # RHEL/CentOS
update-initramfs -u  # Debian/Ubuntu

# ─── Kernel Parameters ─────────────────────────────────────
# View parameters passed to current kernel at boot
cat /proc/cmdline
```

### 🏭 Real-Time Production Use Case

**Scenario**: SaaS company running 500+ EC2 instances, auto-scaling group needs faster node ready time.

- **Problem**: New EC2 nodes were taking 85–110 seconds to boot and be ready to accept Kubernetes pods. During traffic spikes, scale-out was too slow.
- **Investigation**: `systemd-analyze blame` revealed `cloud-init.service` taking 45 seconds waiting for instance metadata, and `update-motd.service` taking 12 seconds running network calls.
- **Solution**:
  1. Disabled `update-motd` dynamic generation
  2. Set `cloud-init` datasource to only check EC2 metadata (not all providers)
  3. Pre-baked AMI with all packages already installed (removed apt update from user-data)
  4. Added `GRUB_CMDLINE_LINUX="quiet splash=0"` to reduce kernel init verbosity
- **Result**: Boot-to-ready time dropped from 95s average to 38s. Auto-scaling response time improved by 60%.

### ❌ Common Mistakes

1. **Editing `/boot/grub2/grub.cfg` directly** — This file is auto-generated. Your changes will be overwritten on the next `update-grub` or kernel update. Always edit `/etc/default/grub` and regenerate.
2. **Forgetting to run `update-initramfs` after adding new hardware drivers** — If you add a new storage controller or add a new kernel module that's needed at early boot, the initramfs must be rebuilt or the system won't boot.
3. **Confusing `systemctl isolate` vs `systemctl set-default`** — `isolate` changes the running target immediately (but reverts on reboot). `set-default` changes the boot target persistently. Use `set-default` for permanent changes.
4. **Not knowing rescue vs emergency mode** — `rescue.target` mounts filesystems and starts basic services. `emergency.target` gives you a shell with only the root filesystem (read-only). For severe filesystem corruption, you need emergency mode.
5. **Removing `quiet` from GRUB without intent** — Removing `quiet` shows all kernel boot messages on screen, which is useful for debugging but significantly slows visual boot on physical servers.

### 🔧 Troubleshooting Tips

```
Problem: System hangs after GRUB, before login
Symptom: Black screen or spinning cursor
Root Cause: Usually a failed systemd service or filesystem mount issue
Fix:
  1. Reboot → at GRUB menu press 'e'
  2. Add to kernel line: systemd.unit=emergency.target
  3. Press Ctrl+X to boot
  4. Run: journalctl -xb --no-pager   # check what failed
  5. Or: systemctl list-units --failed
  6. Fix the failed unit (disable it, fix its config, check dependencies)
```

```
Problem: Kernel panic at boot: "Unable to mount root filesystem"
Symptom: "Kernel panic - not syncing: VFS: Unable to mount root fs"
Root Cause: Initramfs missing disk driver, wrong root= UUID in GRUB, or filesystem corruption
Fix:
  1. Boot from live USB / rescue media
  2. Check: blkid /dev/sda1   # verify UUID matches /boot/grub/grub.cfg
  3. Mount chroot environment:
     mount /dev/sda1 /mnt
     mount --bind /dev /mnt/dev
     mount --bind /proc /mnt/proc
     chroot /mnt
  4. Rebuild initramfs: dracut --force OR update-initramfs -u
  5. Update GRUB: update-grub OR grub2-mkconfig
  6. Reboot
```

```
Problem: Boot is very slow (60+ seconds)
Symptom: `systemd-analyze` shows high userspace time
Root Cause: A service with a 90-second timeout hanging, or dependency deadlock
Fix:
  systemd-analyze blame | head -20         # find slowest services
  systemd-analyze critical-chain           # find the bottleneck
  journalctl -b | grep -i "timeout\|fail"  # look for timeouts
  # Disable or fix the slow service
  systemctl disable slow-service.service
```

```
Problem: "error: no such device" in GRUB
Symptom: GRUB shows error before menu appears
Root Cause: GRUB's device.map or UUID references are stale (after disk replacement/cloning)
Fix:
  1. At GRUB rescue prompt: ls (lists detected devices)
  2. Find the right partition: ls (hd0,1)/   # look for /boot
  3. Set and boot:
     set root=(hd0,1)
     set prefix=(hd0,1)/grub2
     insmod normal; normal
  4. Once booted: reinstall GRUB: grub2-install /dev/sda
```

### 🎤 Interview Questions & Answers

**Q: Walk me through the complete Linux boot process.**
A: The boot process has 6 stages: (1) **BIOS/UEFI** — firmware runs POST, finds the bootable disk, and loads the first-stage bootloader. UEFI is faster and supports GPT disks. (2) **GRUB2** — loads from the EFI partition or MBR, reads `/boot/grub2/grub.cfg`, presents a menu, then loads the kernel image (`vmlinuz`) and initial RAM disk (`initramfs`) into memory, passing kernel parameters. (3) **Kernel initialisation** — the kernel decompresses, initialises hardware, and mounts the initramfs as a temporary root filesystem. (4) **initramfs** — contains minimal drivers and tools needed to access the real root filesystem. It runs udev for device detection, mounts the real `/`, then pivots to it. (5) **systemd (PID 1)** — takes over on the real filesystem, reads unit files, resolves dependencies, and starts all services in parallel up to the default target. (6) **Login** — getty on TTYs and sshd accept logins.

**Q: What is the purpose of initramfs and why is it needed?**
A: The **initramfs** (initial RAM filesystem) solves a chicken-and-egg problem: to mount the root filesystem from a disk, you need disk drivers loaded. But drivers are files on the disk you're trying to mount. The initramfs is a compressed cpio archive loaded into RAM by GRUB before the kernel starts. It contains just enough drivers (storage, filesystem), plus `udev` for device detection, to locate and mount the real root filesystem. It's especially important for systems using LVM, LUKS encryption, RAID, or non-standard storage drivers that aren't compiled directly into the kernel.

**Q: A production server fails to boot after a kernel upgrade. How do you recover it?**
A: First, reboot and select the **previous kernel** from the GRUB menu (GRUB keeps 2-3 kernels). If that works, the new kernel is the problem — investigate with `journalctl -b -1` (logs from previous boot). If no previous kernel works, boot from **GRUB rescue mode**: at the GRUB menu, press `e`, add `systemd.unit=emergency.target` to the kernel line, boot, then diagnose with `journalctl -xb`. If the filesystem is corrupt, boot from a **live/rescue USB**, chroot into the system, and run `fsck` or rebuild the initramfs with `dracut --force`. For cloud instances (AWS EC2), detach the root volume, attach to a rescue instance, chroot, fix, reattach.

**Q: What is systemd-analyze and how do you use it to optimise boot time?**
A: `systemd-analyze` measures boot performance. Key commands: `systemd-analyze` shows total time split between firmware, bootloader, kernel, and userspace. `systemd-analyze blame` lists all services ordered by startup time — use this to identify bottlenecks. `systemd-analyze critical-chain` shows the dependency chain that determined total boot time. `systemd-analyze plot > boot.svg` generates a visual timeline. In production, I target services taking >5 seconds for investigation — common culprits are cloud-init (can be configured to skip non-applicable datasources), NetworkManager-wait-online.service (can be disabled if not needed), and services with missing dependencies causing 90-second timeouts.

**Q: What is the difference between `rescue.target` and `emergency.target` in systemd?**
A: **`rescue.target`** is equivalent to single-user mode — it mounts all local filesystems, starts some basic services (syslog), and gives you a root shell. Network is NOT started. Good for: fixing service configs, running fsck when you can unmount filesystems, password recovery. **`emergency.target`** is more minimal — it mounts only the root filesystem (read-only), starts almost nothing, and gives you an immediate root shell. Use this when: rescue.target itself fails to start, you have severe filesystem corruption that prevents full mount, or you need the earliest possible intervention point. To enter either: add `systemd.unit=rescue.target` or `systemd.unit=emergency.target` to GRUB kernel parameters.

---

## 4. File System Hierarchy (FHS)

### 📖 Concept Explanation

The **Filesystem Hierarchy Standard (FHS)** defines the directory structure and directory contents in Linux. Every file has a defined home based on its purpose. Understanding FHS is non-negotiable for DevOps — you must know where config files, logs, binaries, and data live.

| Directory | Full Name | Purpose |
|---|---|---|
| `/` | Root | Top of the filesystem tree — everything branches from here |
| `/bin` | Binaries | Essential user commands needed in single-user mode (ls, cp, bash) — symlink to /usr/bin on modern systems |
| `/sbin` | System Binaries | Admin commands (fdisk, iptables, sshd) — symlink to /usr/sbin |
| `/usr` | Unix System Resources | Secondary hierarchy — most user utilities and applications |
| `/usr/local` | Local | Locally compiled/installed software (not managed by package manager) |
| `/etc` | Et Cetera | System-wide configuration files — text files, not binaries |
| `/var` | Variable | Variable data: logs (/var/log), mail, spool, packages (/var/lib) |
| `/tmp` | Temporary | Temporary files — cleared on reboot. World-writable with sticky bit |
| `/home` | Home | User home directories — /home/username |
| `/root` | Root Home | Home directory for the root user (not inside /home!) |
| `/opt` | Optional | Third-party software packages (e.g., /opt/google, /opt/splunk) |
| `/proc` | Process | **Virtual FS** — kernel exposes process and system info as files |
| `/sys` | System | **Virtual FS** — exposes kernel device and driver info |
| `/dev` | Devices | **Virtual FS** — device files (block, character, pseudo-devices) |
| `/boot` | Boot | Kernel, initramfs, GRUB files |
| `/lib` | Libraries | Shared libraries needed by /bin and /sbin binaries |
| `/media` | Media | Mount points for removable media (USB, CD) |
| `/mnt` | Mount | Temporary mount points for admins |
| `/run` | Runtime | Runtime data since last boot (PID files, sockets) — tmpfs, lives in RAM |
| `/srv` | Service Data | Data for services (web server document root, FTP) |

### 🎯 Why It Is Used

The FHS prevents chaos. Without it, every application would store configs and data wherever it wanted. With it, you always know: config = `/etc`, logs = `/var/log`, user data = `/home`, system binaries = `/usr/bin`. This is critical for automation — Ansible playbooks, Dockerfiles, and backup scripts all rely on predictable paths.

### 🏗️ Architecture Diagram

```
/  (root filesystem)
├── bin  ──────────────────────── Essential user binaries (ls, cp, cat)
├── sbin ──────────────────────── System admin binaries (fdisk, sshd)
├── usr/
│   ├── bin  ──────────────────── Non-essential user binaries (python, vim)
│   ├── sbin ──────────────────── Non-essential system binaries
│   ├── lib  ──────────────────── Libraries for /usr/bin
│   ├── share ─────────────────── Architecture-independent data, man pages
│   └── local/ ─────────────────── Manually installed software
│       ├── bin/
│       └── lib/
├── etc/ ──────────────────────── System configuration (text files)
│   ├── passwd ─────────────────── User account info
│   ├── fstab ──────────────────── Filesystem mount table
│   ├── ssh/sshd_config ────────── SSH server config
│   └── systemd/ ───────────────── systemd unit files
├── var/
│   ├── log/ ───────────────────── Log files (syslog, auth.log, nginx/)
│   ├── lib/ ───────────────────── Persistent application data (mysql/, docker/)
│   ├── run/ ───────────────────── Runtime PID files
│   └── spool/ ─────────────────── Queued data (mail, cron jobs)
├── home/
│   └── username/ ──────────────── User personal files and dotfiles
├── root/ ──────────────────────── Root user home
├── tmp/ ───────────────────────── Temporary (cleared on reboot)
├── opt/ ───────────────────────── Optional third-party software
├── boot/ ──────────────────────── Kernel, GRUB, initramfs
├── dev/ ───────────────────────── [VIRTUAL] Device files
│   ├── sda, sdb ───────────────── Block devices (disks)
│   ├── tty1..6 ────────────────── Terminal devices
│   ├── null ───────────────────── Discard anything written
│   ├── zero ───────────────────── Infinite null bytes source
│   └── random, urandom ────────── Random number sources
├── proc/ ──────────────────────── [VIRTUAL] Process and kernel info
│   ├── 1/, 2/, ... ────────────── Per-process directories (PID)
│   ├── cpuinfo ────────────────── CPU information
│   ├── meminfo ────────────────── Memory information
│   ├── net/ ───────────────────── Network statistics
│   └── sys/ ───────────────────── Kernel tunable parameters
├── sys/ ───────────────────────── [VIRTUAL] Kernel device tree
│   ├── block/ ─────────────────── Block devices
│   ├── class/ ─────────────────── Device classes
│   └── kernel/ ────────────────── Kernel parameters
└── run/ ───────────────────────── [VIRTUAL] Runtime data (tmpfs)
```

### ⚙️ Key Commands

```bash
# ─── Navigation and inspection ───────────────────────────────
df -h                          # disk usage of all mounted filesystems
du -sh /var/log/*              # size of each item in /var/log
du -sh /* 2>/dev/null | sort -rh | head -10  # find largest directories

# ─── Virtual filesystem inspection ───────────────────────────
cat /proc/cpuinfo              # CPU details (model, cores, flags)
cat /proc/meminfo              # Memory details (MemTotal, MemFree, Cached)
cat /proc/loadavg              # Load average: 1min 5min 15min + running/total procs
cat /proc/net/dev              # Network interface statistics
ls /proc/$(pgrep nginx | head -1)/fd  # file descriptors open by nginx

# Read/write kernel params via /proc/sys
cat /proc/sys/net/ipv4/ip_forward    # check IP forwarding (1=on, 0=off)
echo 1 > /proc/sys/net/ipv4/ip_forward  # enable IP forwarding (temporary)
sysctl -w net.ipv4.ip_forward=1         # same via sysctl (persistent if in sysctl.conf)

# ─── Device files ─────────────────────────────────────────────
ls -la /dev/sd*               # list disk block devices
ls -la /dev/nvme*             # list NVMe devices
cat /dev/urandom | head -c 16 | xxd  # generate 16 random bytes

# ─── Finding config files (FHS-based) ─────────────────────────
find /etc -name "*.conf" -newer /etc/passwd  # configs changed after last user add
ls -la /etc/systemd/system/   # custom systemd units
ls -la /var/log/              # all log files
```

### 🏭 Real-Time Production Use Case

**Scenario**: A Kubernetes cluster's nodes were running out of disk space, causing pods to be evicted.

- **Problem**: `df -h` showed `/` at 95% full on 3 nodes. Pod eviction threshold triggered.
- **Investigation**: `du -sh /* 2>/dev/null | sort -rh` showed `/var` using 180GB. `du -sh /var/* | sort -rh` showed `/var/log` at 40GB and `/var/lib/docker` at 130GB.
- **Root Cause**:
  1. `/var/lib/docker` had orphaned image layers from deployments not being cleaned up
  2. `/var/log/pods` had unbounded log files from a verbose debug-mode app
- **Fix**:
  1. `docker system prune -a --volumes -f` freed 95GB immediately
  2. Added `logrotate` config for pod logs
  3. Set Kubernetes `--image-gc-high-threshold=85` and `--eviction-hard=nodefs.available<10%`
- **Result**: Disk usage stabilised at 40–55%, zero evictions in 60 days.

### ❌ Common Mistakes

1. **Writing application data to `/tmp`** — `/tmp` is cleared on reboot and may be a tmpfs (in RAM). Never store persistent data there. Use `/var/lib/appname` or `/opt/appname/data`.
2. **Installing software directly to `/usr/bin`** — Manually installed binaries should go to `/usr/local/bin`. Files in `/usr/bin` are managed by the package manager and can be overwritten on upgrades.
3. **Confusing `/proc` files with real files** — Don't try to `cp` or `rsync` `/proc` — it's a virtual filesystem. Including `/proc` in backup scripts causes hangs and massive backup sizes.
4. **Storing secrets in `/tmp`** — `/tmp` is world-readable (with sticky bit). Any user can read other users' files in `/tmp` if permissions are loose. Use `mktemp` with mode 0600 for temp secret files.
5. **Filling `/var` on the root partition** — On enterprise setups, mount `/var` as a separate partition. Unbounded log growth can fill `/` causing system instability.

### 🎤 Interview Questions & Answers

**Q: What is the difference between `/proc` and `/sys`?**
A: Both are **virtual filesystems** that expose kernel data as files, but they serve different purposes. `/proc` exposes **process information** (each PID gets its own directory: `/proc/1234/`) and general system stats (`/proc/cpuinfo`, `/proc/meminfo`, `/proc/net/`). It was the original interface. `/sys` (sysfs) was added later and exposes the **kernel device model** — hardware devices, their drivers, and kernel configuration parameters. For example, you configure block device scheduler via `/sys/block/sda/queue/scheduler` and network device info via `/sys/class/net/eth0/`. For tunable kernel parameters, `/proc/sys/` and `sysctl` are the interface.

**Q: Where should application configuration, data, and logs be stored on Linux per FHS?**
A: Per FHS: **Configuration** goes in `/etc/appname/` (system-wide) or `~/.config/appname` (user-specific). **Variable/persistent data** goes in `/var/lib/appname/` — for example, `/var/lib/mysql/` for MySQL data files. **Logs** go in `/var/log/appname/`. **Binaries** from the package manager go in `/usr/bin/` or `/usr/sbin/`. Manually installed binaries go in `/usr/local/bin/`. Third-party bundled software (like Splunk) goes in `/opt/`. Runtime state files (PID files, sockets) go in `/run/appname/`.

**Q: Why is `/run` important and what's special about it?**
A: `/run` is a **tmpfs** filesystem mounted at boot — it lives entirely in RAM and is cleared every reboot. It stores runtime data that needs to be fast and doesn't persist across reboots: PID files (`/run/nginx.pid`), Unix sockets (`/run/docker.sock`), lock files, and runtime state data. Using tmpfs for `/run` means these files are never written to disk, reducing I/O and wear on SSDs. The key insight: if you find stale PID files or sockets after a crash, they're in `/run` and should be cleaned by the service's start script.

---

## 5. Linux Directory & File Operations

### 📖 Concept Explanation

Mastery of file operations is the foundation of Linux productivity. In production you'll navigate complex directory structures, manage thousands of files, and automate operations via scripts. This section covers the essential commands every DevOps engineer uses daily.

### 🎯 Why It Is Used

File operations underpin everything: deployment scripts move artifacts, log analysis pipes files through filters, infrastructure automation creates and modifies config files. Speed and precision with these tools directly impacts how effective you are in production incidents.

### 🏗️ Architecture Diagram

```
FILE OPERATION CATEGORIES:
┌─────────────────────────────────────────────────────────────┐
│                                                               │
│  NAVIGATE        INSPECT         CREATE/MODIFY    SEARCH     │
│  ─────────       ─────────       ─────────────    ──────     │
│  cd              cat             touch            find       │
│  ls              less            mkdir            locate     │
│  pwd             head/tail       cp               grep       │
│  tree            file            mv               which      │
│  pushd/popd      stat            rm               whereis    │
│                  wc              ln (hard/soft)              │
│                                                               │
│  HARD LINK                 SOFT LINK (SYMLINK)               │
│  ─────────                 ──────────────────                │
│  ┌────────┐                ┌────────┐                        │
│  │ name A │──┐             │ name A │──→ /path/to/file       │
│  └────────┘  │──→ inode   └────────┘    (just a pointer)    │
│  ┌────────┐  │             Breaks if target deleted          │
│  │ name B │──┘             Works across filesystems          │
│  └────────┘                                                   │
│  Same inode → same data                                       │
│  Deleting A doesn't affect B                                 │
└─────────────────────────────────────────────────────────────┘
```

### ⚙️ Key Commands

```bash
# ─── Navigation ──────────────────────────────────────────────
pwd                     # print working directory
cd /var/log             # change to absolute path
cd ..                   # go up one level
cd -                    # go to previous directory (very useful!)
ls -la                  # long format, show hidden files
ls -lhS                 # sort by size, human-readable
ls -lt                  # sort by modification time (newest first)
ls -laR /etc/ssl/ 2>/dev/null  # recursive list, suppress permission errors

# ─── File Viewing ─────────────────────────────────────────────
cat /etc/passwd                    # dump entire file to stdout
less /var/log/syslog               # paginate, press q to quit, / to search
head -20 /var/log/nginx/access.log  # first 20 lines
tail -50 /var/log/nginx/error.log   # last 50 lines
tail -f /var/log/syslog             # follow file in real-time (log monitoring)
tail -F /var/log/app.log            # follow and re-open if file rotates (better for logs)

# ─── File Info ─────────────────────────────────────────────────
file /usr/bin/python3           # determine file type (ELF, script, ASCII, etc.)
stat /etc/passwd                # detailed: size, inode, permissions, timestamps
wc -l /etc/passwd               # count lines (39 users = 39 lines)
wc -c /var/log/syslog           # count bytes

# ─── File Operations ───────────────────────────────────────────
cp -av source dest              # copy with verbose + preserve all attributes
cp -r /etc /backup/etc-$(date +%Y%m%d)  # backup /etc with datestamp
mv /tmp/newconfig /etc/app/config  # move (also used for rename)
rm -rf /tmp/build_artifacts     # force recursive delete (BE CAREFUL!)
mkdir -p /opt/myapp/logs/2024   # create full path, no error if exists
touch /var/run/myapp.pid        # create empty file (or update timestamp)
install -m 0755 -o root -g root script.sh /usr/local/bin/  # copy with permissions

# ─── Links ─────────────────────────────────────────────────────
ln /etc/passwd /tmp/passwd_hardlink      # hard link (same inode)
ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/app  # symlink
ls -li /etc/passwd /tmp/passwd_hardlink  # verify same inode number
readlink -f /usr/bin/python3             # resolve full path of symlink chain

# ─── find (production workhorse) ──────────────────────────────
find /var/log -name "*.log" -mtime +30 -delete  # delete logs older than 30 days
find /tmp -type f -size +100M               # files larger than 100MB in /tmp
find /home -type f -name "*.sh" -perm /111  # executable shell scripts
find /etc -name "*.conf" -newer /etc/hosts  # configs modified after /etc/hosts
find / -user www-data -type f 2>/dev/null   # all files owned by www-data
find /var/log -type f -exec ls -lh {} \;    # run ls -lh on each found file
find /opt -name "*.jar" -exec grep -l "log4j" {} \;  # find jars containing log4j
find . -type l -xtype l 2>/dev/null         # find broken symlinks

# ─── which / whereis / locate ─────────────────────────────────
which python3               # shows PATH-resolved location: /usr/bin/python3
whereis python3             # shows binary, source, and man page locations
type ls                     # shows if it's a builtin, alias, or external binary
locate nginx.conf           # fast search using updatedb index (may be stale)
updatedb                    # update locate's database (run as root)
```

### 🏭 Real-Time Production Use Case

**Scenario**: Security audit — finding world-writable files and SUID binaries on 100 servers.

- **Problem**: PCI-DSS audit required identification of all world-writable files and unexpected SUID/SGID binaries.
- **Solution**: Ansible playbook running `find` commands across all hosts:
```bash
# Find unexpected SUID binaries
find / -perm -4000 -type f 2>/dev/null | grep -v -f /etc/allowed_suid_files

# Find world-writable directories (excluding /tmp, /var/tmp)
find / -type d -perm -0002 -not -path "/tmp*" -not -path "/var/tmp*" 2>/dev/null

# Find files modified in last 24h in critical directories
find /etc /usr/bin /usr/sbin -newer /tmp/audit_baseline -type f 2>/dev/null
```
- **Result**: Found 3 unexpected SUID binaries and 12 world-writable directories. All remediated. Passed audit.

### ❌ Common Mistakes

1. **`rm -rf /` or `rm -rf /*`** — The most catastrophic command. Always double-check `rm -rf` commands. Use `--preserve-root` (default on modern systems). Prefer `mv` to a trash directory first.
2. **`cp` without `-a` for backup** — `cp` without flags doesn't preserve timestamps, permissions, or symlinks. Use `cp -a` or `rsync -a` for backups to preserve all attributes.
3. **`find . -exec rm {} \;` without testing first** — Always test `find` with `ls` before using `-exec rm`. Use `find ... -print` to verify what would be deleted.
4. **Confusing hard links and symlinks** — Hard links can only exist on the same filesystem and can't link directories. Symlinks can span filesystems and link directories but break if the target is deleted/moved.
5. **Using `locate` for security auditing** — `locate` uses an index database (`updatedb`). The database may be hours old. For security, always use `find` which searches in real-time.

### 🔧 Troubleshooting Tips

```
Problem: "No space left on device" but df -h shows free space
Symptom: Can't create files, but df shows 20% free
Root Cause: Inode exhaustion — ran out of inodes, not disk space
Fix:
  df -i                    # check inode usage (look for 100% Use%)
  find / -xdev -type f | wc -l   # count total files (slow but accurate)
  # Common cause: millions of tiny files in cache/session dirs
  # Fix: find and delete old files
  find /var/lib/php/sessions -type f -mtime +7 -delete
```

```
Problem: Cannot delete file — "Operation not permitted" even as root
Symptom: rm returns error even with sudo
Root Cause: File has immutable attribute set (chattr +i)
Fix:
  lsattr filename          # shows ----i------- if immutable
  chattr -i filename       # remove immutable flag
  rm filename              # now works
```

```
Problem: Symlink points to wrong location after moving files
Symptom: ls -la shows symlink in red, commands using it fail
Root Cause: Symlinks store the path as-written, which may be relative or have moved
Fix:
  ls -la /the/symlink         # see current target path
  readlink -f /the/symlink    # see resolved absolute path (if broken, shows partial)
  ln -sfn /correct/target /the/symlink  # -f force, -n no-dereference for dirs
```

### 🎤 Interview Questions & Answers

**Q: What is the difference between a hard link and a soft link (symlink)?**
A: A **hard link** is another directory entry pointing to the same inode (the actual data on disk). Both the original and hard link are equal — deleting one doesn't affect the other. The file data is only deleted when the last hard link is removed (link count drops to 0). Limitations: can't cross filesystem boundaries, can't link directories. A **soft link (symlink)** is a special file containing a text pointer to another path. If the target is deleted or moved, the symlink breaks. Symlinks can cross filesystems and can point to directories. In DevOps, symlinks are heavily used: `/etc/nginx/sites-enabled/` symlinks to `sites-available/`, Python version management via `/usr/bin/python3 -> python3.11`.

**Q: How do you find all files modified in the last 24 hours in `/etc`?**
A: `find /etc -type f -mtime -1` — the `-mtime -1` means "modified less than 1 day ago" (note: `-1` means less than 1*24 hours). For more precision: `find /etc -type f -newer /tmp/marker` where `/tmp/marker` is a file you `touch`ed at your baseline time. For very recent changes: `find /etc -type f -mmin -60` finds files modified in the last 60 minutes. This is commonly used in security auditing and change tracking.

**Q: The `find` command is very powerful — give me a production use case with `-exec`.**
A: A common production use case is log cleanup: `find /var/log -name "*.log" -mtime +90 -size +10M -exec gzip {} \;` — this finds log files older than 90 days AND larger than 10MB, then compresses them. Another common one for security: `find / -perm -4000 -type f 2>/dev/null -exec ls -la {} \;` — lists all SUID binaries with their permissions, helping identify privilege escalation risks. The `{}` is a placeholder for each found file, and `\;` terminates the `-exec` expression. For performance with many files, use `+` instead of `\;`: `-exec ls -la {} +` passes multiple files to a single `ls` invocation.

---

# 🔵 SECTION 2 — SHELL & SCRIPTING

---

## 6. Shell Basics & Types

### 📖 Concept Explanation

A **shell** is a command interpreter — it reads commands from the user or a script, interprets them, and passes them to the OS for execution. The shell is the primary interface between humans and the Linux kernel.

Key shell types:
- **bash** (Bourne Again Shell) — Default on most Linux distros. Rich scripting features.
- **sh** (POSIX sh) — Minimal POSIX-compliant shell. May be `dash` on Debian/Ubuntu systems.
- **zsh** — Extended bash with superior tab completion and plugins (used in macOS since Catalina)
- **dash** — Lightweight, fast POSIX shell used as `/bin/sh` on Debian/Ubuntu for boot scripts

**Login vs Non-Login Shells**:
- **Login shell**: Started when you log in (SSH, console). Reads `/etc/profile`, then `~/.bash_profile` (or `~/.profile`)
- **Non-login shell**: Started by opening a terminal in a GUI, or spawning a subshell. Reads `~/.bashrc`

**Interactive vs Non-Interactive**:
- **Interactive**: You type commands, shell prompts for input
- **Non-interactive**: Running a script — no human interaction, fewer startup files loaded

### 🏗️ Architecture Diagram

```
SHELL STARTUP FILES — ORDER OF READING:
┌─────────────────────────────────────────────────────────────────┐
│  SSH Login / Console Login (Login + Interactive Shell)           │
│                                                                   │
│  /etc/profile ──────────────────────────────────────────────    │
│       └──→ /etc/profile.d/*.sh (distribution scripts)           │
│  ~/.bash_profile (or ~/.bash_login, or ~/.profile)               │
│       └──→ usually sources ~/.bashrc                             │
│  ~/.bashrc ──────────────────────────────────────────────────   │
│       └──→ /etc/bashrc (or /etc/bash.bashrc)                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Terminal emulator in GUI / new bash in script                   │
│  (Non-login, Interactive Shell)                                  │
│                                                                   │
│  ~/.bashrc only ────────────────────────────────────────────    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Running a script: bash myscript.sh                             │
│  (Non-login, Non-interactive Shell)                              │
│                                                                   │
│  NOTHING sourced automatically!                                  │
│  (This is why scripts need explicit PATH and env setup)          │
└─────────────────────────────────────────────────────────────────┘

SHELL TYPE COMPARISON:
┌─────────────┬──────────┬──────────┬──────────────────────────┐
│ Shell       │ Speed    │ Features │ Use Case                  │
├─────────────┼──────────┼──────────┼──────────────────────────┤
│ bash 5.x    │ Medium   │ Full     │ Interactive + scripting   │
│ sh (POSIX)  │ Fast     │ Minimal  │ Portable scripts          │
│ dash        │ Fastest  │ POSIX    │ /bin/sh on Debian (boot)  │
│ zsh         │ Medium   │ Most     │ Developer workstations    │
│ fish        │ Medium   │ Modern   │ User-friendly (non-POSIX) │
└─────────────┴──────────┴──────────┴──────────────────────────┘
```

### ⚙️ Key Commands

```bash
# Check current shell
echo $SHELL           # your default shell: /bin/bash
echo $0               # current running shell
ps -p $$              # process info for current shell PID

# Check available shells
cat /etc/shells       # list of valid login shells

# Switch shells
chsh -s /bin/zsh      # change default login shell (requires re-login)
bash                  # start a bash subshell from within any shell
exec bash             # replace current shell with bash (no subshell)

# Identify shell startup
bash -l               # simulate login shell (reads .bash_profile)
bash --norc           # skip .bashrc (useful for clean environments)
bash --noprofile      # skip /etc/profile and .bash_profile

# Check if running interactive
[[ $- == *i* ]] && echo "interactive" || echo "non-interactive"

# Source (execute in current shell context, not subshell)
source ~/.bashrc      # reload bash config
. ~/.bashrc           # same, POSIX syntax — note the dot

# Important: . vs ./
. script.sh           # SOURCE: runs in current shell, variables persist
./script.sh           # EXECUTE: runs in subshell, variables don't persist
bash script.sh        # EXECUTE in new bash: same as above
```

### 🏭 Real-Time Production Use Case

**Scenario**: Debugging why a cron job "works when run manually but fails in cron."

The infamous problem: scripts that work interactively but fail in cron. This is almost always an environment issue. Cron runs scripts with a minimal non-interactive, non-login shell. `$PATH` is typically just `/usr/bin:/bin`. If your script calls `/usr/local/bin/python3` or `~/go/bin/somebinary`, cron can't find it.

**Fix pattern**:
```bash
#!/bin/bash
# At the top of any cron script — define explicit PATH
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Or source the user's environment:
source /home/ubuntu/.bashrc    # loads PATH and custom env vars
```

### ❌ Common Mistakes

1. **Using `#!/bin/sh` with bash-specific syntax** — If your script uses arrays `${myarray[@]}`, `[[ ]]`, or `(())`, you MUST use `#!/bin/bash`. On Debian/Ubuntu, `/bin/sh` is `dash` which doesn't support these bash extensions.
2. **Putting PATH exports in `~/.bashrc` but expecting them in scripts** — Scripts run in non-interactive shells that don't source `~/.bashrc`. Always set PATH explicitly in scripts.
3. **Forgetting `.bash_profile` vs `.bashrc` distinction** — In SSH sessions (login shell), `~/.bashrc` is NOT automatically read unless `~/.bash_profile` explicitly sources it. Symptom: aliases work in GUI terminals but not in SSH.
4. **Modifying `/etc/profile` directly** — Don't edit `/etc/profile` directly. Add scripts to `/etc/profile.d/` instead — it's included by `/etc/profile` and your changes survive OS upgrades.

### 🎤 Interview Questions & Answers

**Q: What is the difference between `.bashrc` and `.bash_profile`?**
A: **`.bash_profile`** is read by **login shells** — when you SSH into a server or log in on a console. It's read once per session. **`.bashrc`** is read by **interactive non-login shells** — when you open a new terminal tab or run `bash` from within a session. For consistency, it's best practice to have `~/.bash_profile` source `~/.bashrc` so configuration is always loaded. Rule of thumb: put environment variables (`export PATH=...`, `export JAVA_HOME=...`) in `.bash_profile` (or `.profile` for portability); put aliases, functions, and prompt settings in `.bashrc`.

**Q: What is the difference between `source script.sh` and `./script.sh`?**
A: `source script.sh` (or `. script.sh`) runs the script in the **current shell process** — any variables set, directory changes (`cd`), or environment modifications persist after it finishes. `./script.sh` starts a **new subprocess** — a child shell. Any variables or `cd` commands in the script only affect the child and disappear when it exits. This is why environment setup scripts (like `activate` in Python virtualenvs or `~/.bashrc`) must be sourced, not executed.

---

## 7. Shell Features & Productivity

### 📖 Concept Explanation

Mastering shell features transforms you from someone who uses the command line into someone who commands it. These features are the difference between taking 10 minutes to do something and 30 seconds.

### ⚙️ Key Commands

```bash
# ─── Variables ────────────────────────────────────────────────
NAME="devops"                   # assign (no spaces around =)
echo $NAME                      # use variable
echo "${NAME}_engineer"         # brace syntax for disambiguation
unset NAME                      # remove variable

# ─── Environment Variables ───────────────────────────────────
export DATABASE_URL="postgres://localhost:5432/mydb"  # make available to child processes
printenv                        # list all environment variables
env | grep JAVA                 # find Java-related env vars
export -f myfunc                # export a function to child processes

# ─── Command Substitution ─────────────────────────────────────
CURRENT_DIR=$(pwd)              # modern syntax (preferred, nestable)
CURRENT_DIR=`pwd`               # old backtick syntax (avoid — can't nest)
FILES=$(find /etc -name "*.conf" | wc -l)
echo "Found $FILES config files"

# ─── Aliases ──────────────────────────────────────────────────
alias ll='ls -lah'              # short alias
alias gs='git status'
alias k='kubectl'               # common K8s shortcut
alias ..='cd ..'
alias grep='grep --color=auto'  # always colorise grep output
unalias ll                      # remove alias
alias                           # list all defined aliases

# ─── History ──────────────────────────────────────────────────
history | tail -20              # last 20 commands
!!                              # repeat last command
!vim                            # repeat last command starting with 'vim'
!$                              # last argument of previous command
Ctrl+R                          # reverse search through history (most useful!)
HISTSIZE=10000                  # number of commands to keep in memory
HISTFILESIZE=20000              # number of lines in ~/.bash_history
HISTCONTROL=ignoredups:erasedups  # don't store duplicates

# ─── Job Control ──────────────────────────────────────────────
sleep 100 &                     # run in background, returns immediately
jobs                            # list background jobs: [1] Running sleep 100
fg %1                           # bring job 1 to foreground
bg %1                           # send stopped job to background
Ctrl+Z                          # pause (stop) foreground job
Ctrl+C                          # terminate foreground job (SIGINT)

nohup ./long_script.sh &        # run immune to hangup (survives logout)
nohup ./script.sh > /tmp/output.log 2>&1 &  # with output captured
disown %1                       # remove job from job table (won't be killed on logout)
screen -dmS mysession command   # run in screen session (survives SSH disconnect)
tmux new-session -d -s work     # tmux detached session

# ─── I/O Redirection ──────────────────────────────────────────
command > file.txt              # redirect stdout to file (overwrite)
command >> file.txt             # append stdout to file
command < input.txt             # read stdin from file
command 2> errors.txt           # redirect stderr to file
command 2>&1                    # redirect stderr to same place as stdout
command > output.txt 2>&1       # both stdout and stderr to file
command &> output.txt           # bash shorthand for above
command 2>/dev/null             # discard stderr (suppress error messages)
command >/dev/null 2>&1         # discard all output (run silently)

# ─── Here Documents (heredoc) ─────────────────────────────────
cat << EOF > /etc/myapp/config.yaml
database:
  host: ${DB_HOST}
  port: 5432
  name: myapp
EOF

# Heredoc in script (no variable expansion with 'EOF' in quotes)
cat << 'EOF'
This $VARIABLE won't be expanded
EOF
```

### 🏭 Real-Time Production Use Case

Using I/O redirection and job control together for a production deployment:
```bash
# Deploy in background, capture all output, monitor progress
nohup ./deploy.sh --env=prod 2>&1 | tee /var/log/deployments/$(date +%Y%m%d_%H%M%S).log &
echo "Deployment PID: $!"       # $! = PID of last background process
tail -f /var/log/deployments/*.log  # follow the latest log
```

### ❌ Common Mistakes

1. **`VAR = "value"`** — Spaces around `=` in variable assignment cause a syntax error. The shell interprets `VAR` as a command with ` = "value"` as arguments. Always: `VAR="value"` (no spaces).
2. **`command 2>&1 >file`** — Wrong order! This sends stderr to wherever stdout is pointing at that moment (likely the terminal), THEN redirects stdout to file. Correct order: `command >file 2>&1`.
3. **Not quoting variables** — `rm -rf $MYVAR/*` — if `$MYVAR` is empty, this becomes `rm -rf /*`. Always quote: `rm -rf "${MYVAR}/"*`.
4. **Using `export` unnecessarily inside scripts** — `export` is only needed to make a variable available to child processes. For variables used only within a script, plain assignment is sufficient and cleaner.

### 🎤 Interview Questions & Answers

**Q: What does `2>&1` mean and why does the order matter?**
A: `2>&1` means "redirect file descriptor 2 (stderr) to wherever file descriptor 1 (stdout) is currently pointing." The order critically matters because redirections are processed left-to-right. `command > file 2>&1` means: (1) redirect stdout to `file`; (2) redirect stderr to where stdout now points (which is `file`). So both go to `file`. But `command 2>&1 > file` means: (1) redirect stderr to where stdout currently points (the terminal); (2) then redirect stdout to `file`. Result: stderr goes to terminal, stdout goes to file — not what you typically want.

---

## 8. Pipes & Filters (Text Processing Masterclass)

### 📖 Concept Explanation

The Unix philosophy: "Do one thing well, and chain tools together." Pipes (`|`) connect stdout of one command to stdin of the next. This enables building powerful one-liners from simple tools.

**Core text processing tools**:
- **grep**: Pattern matching — find lines that match a regex
- **sed**: Stream editor — transform text (find/replace, delete, insert)
- **awk**: Field-based processing — extract columns, compute sums, generate reports
- **cut**: Extract specific columns/fields
- **sort**: Sort lines
- **uniq**: Remove/count duplicate lines (requires sorted input)
- **wc**: Count lines, words, bytes
- **tr**: Translate/replace characters

### 🏗️ Architecture Diagram

```
PIPE CHAINING:
         stdout        stdin          stdout        stdin
┌───────┐  │  ┌───┐  │  ┌─────────┐  │  ┌───┐  │  ┌────────┐
│command│──┼─▶│ | │──┼─▶│ command │──┼─▶│ | │──┼─▶│command │
└───────┘     └───┘     └─────────┘     └───┘     └────────┘
stderr not         stderr not
piped (goes to     piped (goes to
 terminal)          terminal)

TEE - splits the stream:
┌──────────┐       ┌─────┐ ──→ stdout (terminal/next pipe)
│ command  │──────▶│ tee │
└──────────┘       └─────┘ ──→ file.txt (copy to file)
```

### ⚙️ Key Commands

```bash
# ─── grep ─────────────────────────────────────────────────────
grep "error" /var/log/syslog                # basic pattern match
grep -i "error" /var/log/syslog             # case-insensitive
grep -r "password" /etc/ 2>/dev/null        # recursive search in directory
grep -v "DEBUG" app.log                     # invert match (exclude lines)
grep -E "ERROR|WARN|CRIT" app.log           # extended regex (or)
grep -n "failed" auth.log                   # show line numbers
grep -c "404" access.log                    # count matching lines
grep -A 3 -B 3 "OutOfMemoryError" app.log   # 3 lines After and Before match
grep -l "api_key" /etc/**/*.conf            # only show filenames with matches
grep -P "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" access.log  # Perl regex (IP addresses)

# ─── sed ──────────────────────────────────────────────────────
sed 's/old/new/' file.txt               # replace first occurrence per line
sed 's/old/new/g' file.txt              # replace ALL occurrences per line
sed -i 's/localhost/db.prod.internal/g' /etc/app/config.ini  # in-place edit!
sed -i.bak 's/foo/bar/g' file.txt       # in-place with backup (.bak)
sed -n '10,20p' file.txt                # print lines 10 to 20
sed '/^#/d' config.conf                 # delete comment lines
sed 's/^/PREFIX: /' file.txt            # add prefix to every line
sed 's/[[:space:]]*$//' file.txt        # strip trailing whitespace

# ─── awk ──────────────────────────────────────────────────────
awk '{print $1}' file.txt               # print first field (space-delimited)
awk -F: '{print $1}' /etc/passwd        # print usernames (: delimiter)
awk '{print $NF}' file.txt              # print last field
awk '{sum += $3} END {print sum}' data.txt  # sum column 3
awk '/ERROR/ {print $0}' app.log        # print lines matching pattern
awk '$5 > 100 {print $1, $5}' data.txt  # conditional: print if col5 > 100
awk 'NR==5' file.txt                    # print line 5 only
awk -F: '$3 >= 1000 {print $1}' /etc/passwd  # print usernames with UID >= 1000 (regular users)

# Nginx log analysis with awk:
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20
# ^ top 20 most requested URLs

awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
# ^ count HTTP response codes

# ─── cut ──────────────────────────────────────────────────────
cut -d: -f1,3 /etc/passwd           # fields 1 and 3, colon delimiter (username + UID)
cut -d',' -f2 data.csv              # second CSV field
cut -c1-10 file.txt                 # first 10 characters of each line

# ─── sort & uniq ──────────────────────────────────────────────
sort file.txt                       # alphabetical sort
sort -n numbers.txt                 # numerical sort
sort -rn numbers.txt                # reverse numerical sort
sort -t: -k3 -n /etc/passwd         # sort by 3rd field (UID), colon delimiter
sort -u file.txt                    # sort and remove duplicates
sort file.txt | uniq -c | sort -rn  # count occurrences, show most common first
sort file.txt | uniq -d             # show only duplicate lines

# ─── tr ───────────────────────────────────────────────────────
tr 'a-z' 'A-Z' < file.txt          # convert to uppercase
tr -d '\r' < windows.txt > unix.txt  # remove Windows carriage returns
tr -s ' ' < file.txt                # squeeze multiple spaces into one
echo "hello world" | tr ' ' '_'    # replace spaces with underscores

# ─── tee ──────────────────────────────────────────────────────
command | tee output.log            # output to screen AND file simultaneously
command | tee -a output.log         # append to file
command 2>&1 | tee all_output.log   # capture all output to file and screen

# ─── xargs ────────────────────────────────────────────────────
find /tmp -name "*.log" | xargs rm -f          # delete found files
find /etc -name "*.conf" | xargs grep "timeout"  # search in found files
echo "file1 file2 file3" | xargs -n1 touch    # create each file (one per run)
cat servers.txt | xargs -P5 -I{} ssh {} uptime  # parallel SSH (5 at a time)

# ─── Production one-liners ────────────────────────────────────
# Top 10 IP addresses in nginx access log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Count HTTP status codes
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Find all failed SSH login attempts and count by IP
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head -20

# Extract all unique email addresses from a file
grep -oP '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file.txt | sort -u

# Find the 10 largest files in /var
find /var -type f -printf '%s %p\n' 2>/dev/null | sort -rn | head -10 | awk '{printf "%s\t%s\n", $1/1024/1024 "MB", $2}'

# Monitor error rate per minute in real-time
tail -f /var/log/app.log | awk '/ERROR/ {count++} /^[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}/ {if(count>0){print $1" "$2": "count" errors"; count=0}}'

# Check if any process is using more than 1GB of RAM
ps aux | awk '{if ($6 > 1048576) print $0}' | sort -k6 -rn
```

### 🎤 Interview Questions & Answers

**Q: Given an nginx access log, how would you find the top 10 IP addresses making requests?**
A:
```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```
Explanation: `awk '{print $1}'` extracts the first field (IP address). `sort` groups identical IPs together (required before `uniq`). `uniq -c` counts consecutive identical lines. `sort -rn` sorts numerically in reverse (highest count first). `head -10` shows top 10.

**Q: What's the difference between grep, sed, and awk? When do you use each?**
A: **grep** is for **filtering lines** — finding lines that match a pattern. It doesn't transform, just selects. Use it when you want to find or count lines. **sed** is a **stream editor** for transformations — find and replace, delete lines, insert text. Best for simple text substitutions. **awk** is a full **field-based processing language** — it splits lines into fields, supports variables, arithmetic, conditionals, and can generate formatted reports. Use awk when grep/sed aren't powerful enough: summing a column, processing CSV, conditional logic on fields.

---

## 9. Bash Scripting (Production Grade)

### 📖 Concept Explanation

Production bash scripts are different from tutorial scripts. They handle errors gracefully, are idempotent, have proper logging, and won't destroy things if they fail halfway through.

### 🏗️ Architecture Diagram

```
PRODUCTION BASH SCRIPT STRUCTURE:
┌─────────────────────────────────────────────────────────────┐
│  #!/bin/bash                          ← shebang              │
│  set -euo pipefail                    ← safety settings      │
│                                                               │
│  ── Constants / Config ─────────────────────────────────    │
│  readonly SCRIPT_DIR LOG_FILE etc.                           │
│                                                               │
│  ── Logging Functions ──────────────────────────────────    │
│  log_info(), log_error(), log_warn()                         │
│                                                               │
│  ── Utility Functions ──────────────────────────────────    │
│  check_dependencies(), require_root(), etc.                  │
│                                                               │
│  ── Cleanup Trap ───────────────────────────────────────    │
│  trap cleanup EXIT SIGINT SIGTERM                             │
│                                                               │
│  ── Core Functions ─────────────────────────────────────    │
│  main() { parse_args; validate; execute; }                   │
│                                                               │
│  ── Entry Point ────────────────────────────────────────    │
│  main "$@"                            ← always last          │
└─────────────────────────────────────────────────────────────┘
```

### ⚙️ Key Commands & Patterns

```bash
#!/bin/bash
# ─── Safety Settings ──────────────────────────────────────────
set -e          # exit immediately if any command fails
set -u          # treat unset variables as errors
set -o pipefail # pipeline fails if ANY command in it fails (not just last)
set -euo pipefail  # all three combined (gold standard)

# ─── Script metadata ──────────────────────────────────────────
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"
readonly TIMESTAMP="$(date +%Y%m%d_%H%M%S)"

# ─── Logging functions ────────────────────────────────────────
log_info()  { echo "[INFO]  $(date '+%Y-%m-%d %H:%M:%S') $*" | tee -a "$LOG_FILE"; }
log_warn()  { echo "[WARN]  $(date '+%Y-%m-%d %H:%M:%S') $*" | tee -a "$LOG_FILE" >&2; }
log_error() { echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') $*" | tee -a "$LOG_FILE" >&2; }

# ─── Variables & Arrays ───────────────────────────────────────
NAME="world"
COUNTER=0
SERVERS=("web01" "web02" "db01")      # indexed array
declare -A CONFIG                      # associative array (dictionary)
CONFIG["env"]="production"
CONFIG["port"]="8080"

echo "${SERVERS[0]}"                   # first element: web01
echo "${SERVERS[@]}"                   # all elements
echo "${#SERVERS[@]}"                  # count: 3
echo "${CONFIG["env"]}"                # production

# ─── Conditionals ─────────────────────────────────────────────
if [[ -f "/etc/myapp/config.yaml" ]]; then
    log_info "Config found"
elif [[ -d "/etc/myapp" ]]; then
    log_warn "Config directory exists but no config file"
else
    log_error "Neither config nor directory found"
    exit 1
fi

# Common test flags:
# -f  file exists and is regular file
# -d  directory exists
# -z  string is empty
# -n  string is non-empty
# -r  file is readable
# -x  file is executable
# -eq, -ne, -lt, -gt  numeric comparisons
# ==, !=, <, >  string comparisons (inside [[ ]])

# ─── Case Statement ────────────────────────────────────────────
case "$ENVIRONMENT" in
    production|prod)
        DB_HOST="db.prod.internal"
        ;;
    staging|stage)
        DB_HOST="db.staging.internal"
        ;;
    development|dev)
        DB_HOST="localhost"
        ;;
    *)
        log_error "Unknown environment: $ENVIRONMENT"
        exit 1
        ;;
esac

# ─── Loops ────────────────────────────────────────────────────
# For loop over array
for server in "${SERVERS[@]}"; do
    echo "Checking $server"
    ssh "$server" "uptime" || log_warn "$server unreachable"
done

# For loop over command output
for pid in $(pgrep nginx); do
    echo "Nginx PID: $pid"
done

# While loop with counter
while [[ $COUNTER -lt 10 ]]; do
    echo "Attempt $((COUNTER + 1))"
    COUNTER=$((COUNTER + 1))
    sleep 1
done

# Read file line by line (safer than for loop)
while IFS= read -r line; do
    echo "Processing: $line"
done < /etc/hosts

# ─── Functions ────────────────────────────────────────────────
check_dependencies() {
    local deps=("curl" "jq" "git")    # local variable — only visible in function
    for dep in "${deps[@]}"; do
        if ! command -v "$dep" &>/dev/null; then
            log_error "Required dependency missing: $dep"
            return 1                  # return (not exit) from function
        fi
    done
    return 0
}

# ─── Error Handling with trap ──────────────────────────────────
cleanup() {
    local exit_code=$?
    log_info "Script exiting with code: $exit_code"
    # Clean up temp files, release locks, send notifications
    rm -f /tmp/${SCRIPT_NAME}.lock
    if [[ $exit_code -ne 0 ]]; then
        log_error "Script failed! Check $LOG_FILE for details"
        # Send alert: curl -X POST webhook_url -d "Deploy failed"
    fi
}

trap cleanup EXIT        # runs on any exit
trap 'log_error "Caught SIGINT"; exit 1' SIGINT   # Ctrl+C
trap 'log_error "Caught SIGTERM"; exit 1' SIGTERM  # kill command

# ─── Argument Parsing ─────────────────────────────────────────
# Simple positional args
ENVIRONMENT="${1:-development}"     # $1 with default value "development"
APP_NAME="${2:?Error: APP_NAME required}"  # $2 required — exit if missing

# Using getopts (short options)
while getopts "e:n:vh" opt; do
    case $opt in
        e) ENVIRONMENT="$OPTARG" ;;
        n) APP_NAME="$OPTARG" ;;
        v) VERBOSE=true ;;
        h) usage; exit 0 ;;
        *) usage; exit 1 ;;
    esac
done

# Special variables:
# $0  script name
# $1..$9  positional arguments
# $@  all arguments (as separate words) — use this for iteration
# $*  all arguments (as single string)
# $#  count of arguments
# $?  exit code of last command
# $$  PID of current shell
# $!  PID of last background command

# ─── Script Debugging ─────────────────────────────────────────
bash -n script.sh       # syntax check only (dry run)
bash -x script.sh       # execution trace (prints each command before running)
set -x                  # enable trace mode at current point in script
set +x                  # disable trace mode
bash -xv script.sh      # verbose + trace (most detailed debugging)
```

### 🏭 Real-Time Production Use Case

**A production deployment script pattern used by a fintech company for zero-downtime deploys:**

```bash
#!/bin/bash
set -euo pipefail

readonly DEPLOY_DIR="/opt/myapp"
readonly RELEASE_DIR="${DEPLOY_DIR}/releases/$(date +%Y%m%d_%H%M%S)"
readonly CURRENT_LINK="${DEPLOY_DIR}/current"
readonly MAX_RELEASES=5

log_info() { echo "[$(date '+%T')] $*"; }

cleanup_old_releases() {
    log_info "Cleaning old releases (keeping last ${MAX_RELEASES})"
    ls -dt "${DEPLOY_DIR}"/releases/*/ | tail -n +$((MAX_RELEASES + 1)) | xargs rm -rf
}

deploy() {
    local version="$1"
    log_info "Deploying version ${version} to ${RELEASE_DIR}"
    mkdir -p "$RELEASE_DIR"
    
    # Download artifact
    aws s3 cp "s3://my-artifacts/${version}/app.tar.gz" /tmp/app.tar.gz
    tar -xzf /tmp/app.tar.gz -C "$RELEASE_DIR"
    
    # Symlink shared config (secrets)
    ln -s "${DEPLOY_DIR}/shared/config.yaml" "${RELEASE_DIR}/config.yaml"
    
    # Run pre-deploy checks
    "${RELEASE_DIR}/bin/myapp" --check-config || { log_error "Config check failed"; exit 1; }
    
    # Atomic switch (ln -sfn is atomic)
    ln -sfn "$RELEASE_DIR" "$CURRENT_LINK"
    log_info "Switched current to ${RELEASE_DIR}"
    
    # Graceful reload (not restart — zero downtime)
    systemctl reload myapp.service
    log_info "Service reloaded"
    
    cleanup_old_releases
}

deploy "${1:?Usage: $0 <version>}"
```

### ❌ Common Mistakes

1. **Not using `set -euo pipefail`** — Without this, your script continues happily after failures. `VAR=$(failing-command)` won't trigger `set -e` because command substitution swallows exit codes. Explicit checks are still needed for critical commands.
2. **`for item in $(cat file)`** — This breaks on filenames with spaces. Use `while IFS= read -r line; do ... done < file` instead.
3. **Not quoting `"$@"`** — Using `$@` unquoted loses argument grouping. Always `"$@"` when forwarding arguments to preserve original quoting.
4. **Using `exit` in functions** — `exit` terminates the entire script, not just the function. Use `return` with an exit code inside functions.
5. **No error handling on critical commands** — Don't assume commands succeed. Always check: `if ! cp source dest; then log_error "Copy failed"; exit 1; fi`.

### 🎤 Interview Questions & Answers

**Q: What does `set -euo pipefail` do and why is it important?**
A: These are bash safety options: `set -e` makes the script exit immediately if any command returns a non-zero exit code (instead of continuing with corrupt state). `set -u` treats references to undefined variables as errors (catches typos like `${DIRECOTRY}` instead of `${DIRECTORY}`). `set -o pipefail` makes a pipeline fail if ANY command in it fails, not just the last one — without this, `failing_command | tee log.txt` would succeed because `tee` succeeds. Together, these prevent scripts from silently continuing after errors and causing cascading failures.

**Q: How do you handle cleanup in bash scripts when a script fails or is interrupted?**
A: Use `trap`. The pattern is: define a cleanup function that removes temp files, releases locks, and sends failure notifications. Then register it with `trap cleanup EXIT`. The `EXIT` trap runs regardless of how the script exits — normal completion, `exit` call, or error (when combined with `set -e`). You can also trap specific signals: `trap 'cleanup; exit 130' SIGINT` for Ctrl+C. Inside the cleanup function, check `$?` to know if the script succeeded or failed and take different actions.

---

## 10. Regular Expressions (Regex)

### 📖 Concept Explanation

**Regular expressions** are patterns for matching text. They're used everywhere: `grep`, `sed`, `awk`, Python, Nginx configs, Java, Go. In DevOps, regex is used daily for log parsing, validation, and text transformation.

| Regex Type | Used In | Key Difference |
|---|---|---|
| **BRE** (Basic) | `grep`, `sed` default | `+`, `?`, `\|` need escaping: `\+` `\?` `\|` |
| **ERE** (Extended) | `grep -E`, `egrep`, `awk` | `+`, `?`, `|` are special without escaping |
| **PCRE** (Perl-Compatible) | `grep -P`, Python, most languages | `\d`, `\w`, lookahead/lookbehind, named groups |

### ⚙️ Key Commands & Patterns

```bash
# ─── Character Classes ─────────────────────────────────────────
.           # any single character (except newline)
[abc]       # character class: a, b, or c
[a-z]       # range: lowercase a to z
[A-Z0-9]    # uppercase letters or digits
[^abc]      # negated: NOT a, b, or c
\d          # digit: [0-9]          (PCRE: grep -P)
\w          # word char: [a-zA-Z0-9_]
\s          # whitespace: [ \t\n\r]

# ─── Quantifiers ──────────────────────────────────────────────
*           # 0 or more
+           # 1 or more (ERE/PCRE)
?           # 0 or 1 (optional)
{3}         # exactly 3
{3,}        # 3 or more
{3,6}       # 3 to 6

# ─── Anchors ───────────────────────────────────────────────────
^           # start of line
$           # end of line
\b          # word boundary (PCRE)
\B          # non-word boundary

# ─── Groups & Alternation ─────────────────────────────────────
(abc)       # capture group
(?:abc)     # non-capturing group (PCRE)
a|b         # alternation: a or b (ERE/PCRE)

# ─── Practical Regex Patterns ─────────────────────────────────

# IPv4 address
grep -P '\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b' file.txt

# Email address (simplified)
grep -P '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file.txt

# ISO 8601 timestamp (2024-03-15T10:30:00)
grep -P '\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}' logs.txt

# HTTP response codes (4xx or 5xx)
grep -E ' [45][0-9]{2} ' access.log

# Extract values from key=value pairs
grep -oP '(?<=key=)[^\s]+' config.txt

# Lines with 3+ consecutive failed attempts
grep -E '(failed.+){3,}' auth.log

# sed: extract just the IP from "Client IP: 192.168.1.100 connected"
echo "Client IP: 192.168.1.100 connected" | sed 's/.*IP: \([0-9.]*\).*/\1/'
# With ERE: sed -E 's/.*IP: ([0-9.]+).*/\1/'

# awk: extract timestamp column matching pattern
awk '/ERROR/ {match($0, /[0-9]{2}:[0-9]{2}:[0-9]{2}/); print substr($0, RSTART, RLENGTH)}' app.log

# ─── Real-world log parsing ────────────────────────────────────
# Extract all 500 errors with timestamps from nginx log
grep -P '" 500 ' access.log | awk '{print $4}' | tr -d '[' | cut -d: -f1 | sort | uniq -c

# Count unique IPs that got 429 (rate limited)
awk '$9 == 429 {print $1}' access.log | sort -u | wc -l

# Find Java stack trace beginnings (lines starting with Exception or at)
grep -E '^(Exception|Error|	at )' app.log | head -50
```

### 🎤 Interview Questions & Answers

**Q: Write a regex to extract all IPv4 addresses from a log file.**
A:
```bash
grep -oP '\b(?:\d{1,3}\.){3}\d{1,3}\b' logfile.txt
```
The `-o` flag prints only the matching part (not the whole line). `-P` enables PCRE. The pattern `(?:\d{1,3}\.){3}\d{1,3}` matches three groups of 1-3 digits followed by a dot, then a final group of 1-3 digits. `\b` is a word boundary to prevent matching partial IPs within larger numbers. For strict validation of valid IP ranges (0-255), the regex becomes much more complex and a scripting language is preferable.

**Q: What's the difference between `grep`, `grep -E`, and `grep -P`?**
A: `grep` uses **Basic Regular Expressions (BRE)** — special characters like `+`, `?`, `|`, `(`, `)` must be escaped with backslash to have their special meaning. `grep -E` (or `egrep`) uses **Extended Regular Expressions (ERE)** — `+`, `?`, `|`, `()` are special by default without escaping. `grep -P` uses **Perl-Compatible Regular Expressions (PCRE)** — the most powerful option: supports `\d`, `\w`, `\s`, lookaheads `(?=...)`, lookbehinds `(?<=...)`, and non-greedy `*?`. For production scripting, I default to `grep -E` for portability and `grep -P` when I need `\d`/`\w` or lookarounds.

---

# 🟠 SECTION 3 — FILE PERMISSIONS & USER MANAGEMENT

---

## 11. File Permissions & Ownership

### 📖 Concept Explanation

Linux uses a discretionary access control (DAC) model. Every file and directory has:
- **Owner** (user): The user who owns the file
- **Group**: A group of users who share access
- **Other**: Everyone else

Permissions are set for each of these three categories: **read (r=4)**, **write (w=2)**, **execute (x=1)**.

```
ls -la /etc/passwd
-rw-r--r--  1  root  root  2847  Jan 15 09:23  /etc/passwd
│││││││││││  │  ││││  ││││
│││││││││││  │  ││││  └─── group name
│││││││││││  │  └─────── owner name
│││││││││││  └────────── hard link count
│││││││││││
││││├────── other: r-- = 4 (read only)
│├───────── group: r-- = 4 (read only)
└────────── owner: rw- = 6 (read+write)
            type: - = regular file
                  d = directory
                  l = symlink
                  c = character device
                  b = block device
```

**Special Permissions**:
- **SUID (Set User ID)** on executable: runs as the file's owner, not the caller. E.g., `/usr/bin/passwd` — lets regular users change their password (owned by root, has SUID).
- **SGID (Set Group ID)** on executable: runs as file's group. On directories: new files inherit the directory's group.
- **Sticky bit** on directory: only the file owner (or root) can delete files, even if others have write access. Classic use: `/tmp`.

### 🏗️ Architecture Diagram

```
PERMISSION BIT LAYOUT (9 bits + 3 special bits = 12 total):

Full octal: 0755 = --- rwx r-x r-x
             │    │││ ├──── user  = 7 = rwx
             │    │││ ├──── group = 5 = r-x
             │    │││ └──── other = 5 = r-x
             └─── special bits: 0 = none
                  SUID=4, SGID=2, Sticky=1

SPECIAL PERMISSION EXAMPLES:
/usr/bin/passwd:  -rwsr-xr-x  (s = SUID set, runs as root)
/usr/bin/crontab: -rwxr-sr-x  (s = SGID set, runs as crontab group)
/tmp:             drwxrwxrwt  (t = sticky bit, only owner can delete)

PERMISSION DECISION FLOW:
  Am I the file owner? ─── YES ──→ Apply OWNER permissions
         │
         NO
         │
  Is my GID the file group? ─── YES ──→ Apply GROUP permissions
         │
         NO
         │
  Apply OTHER permissions
```

### ⚙️ Key Commands

```bash
# ─── View permissions ─────────────────────────────────────────
ls -la /etc/shadow           # -rw-r----- root shadow = only root + shadow group
stat /etc/ssh/sshd_config    # detailed info including octal permissions

# ─── chmod ────────────────────────────────────────────────────
chmod 644 file.txt           # rw-r--r-- (octal)
chmod 755 script.sh          # rwxr-xr-x (octal) — typical executable
chmod 700 ~/.ssh             # rwx------ (only owner)
chmod 600 ~/.ssh/authorized_keys  # rw------- (only owner read/write)
chmod u+x script.sh          # add execute for user (symbolic)
chmod g-w file.txt           # remove write for group (symbolic)
chmod o=r file.txt           # set other to read only (symbolic)
chmod -R 750 /opt/myapp      # recursive: rwxr-x--- on directory tree
chmod a+r file.txt           # add read for ALL (user+group+other)

# ─── SUID / SGID / Sticky ─────────────────────────────────────
chmod u+s /usr/local/bin/myapp   # set SUID
chmod g+s /data/shared           # set SGID on directory (shared dir)
chmod +t /tmp/mydir              # set sticky bit
chmod 4755 /usr/local/bin/myapp  # SUID + rwxr-xr-x (4=SUID)
chmod 2775 /data/team/           # SGID + rwxrwxr-x (2=SGID)
chmod 1777 /tmp                  # sticky + rwxrwxrwx (1=sticky)

# Find all SUID binaries (security audit)
find / -perm -4000 -type f 2>/dev/null | sort

# ─── chown / chgrp ────────────────────────────────────────────
chown alice file.txt          # change owner to alice
chown alice:developers file.txt  # change owner + group
chown -R www-data:www-data /var/www/html/  # recursive
chgrp docker /var/run/docker.sock  # add docker group access to socket

# ─── umask ────────────────────────────────────────────────────
umask                         # show current umask: e.g., 0022
# umask 0022 means: new files get 0666-0022=0644, new dirs get 0777-0022=0755
umask 0027                    # stricter: files=0640, dirs=0750
# Set permanently in /etc/profile or ~/.bashrc

# ─── ACLs ────────────────────────────────────────────────────
# Standard permissions only allow one owner, one group — ACLs give more granularity
getfacl /var/www/html          # view ACL
setfacl -m u:alice:rx /var/www/html     # give alice read+execute
setfacl -m g:developers:rwx /var/data  # give developers group full access
setfacl -m o::--- /secret/dir          # remove all other permissions
setfacl -b /var/www/html        # remove all ACL entries
setfacl -R -m u:jenkins:rx /opt/app   # recursive ACL for Jenkins user
# Check if filesystem mounted with ACL support (check /etc/fstab for "acl" option)
# Most modern systems mount with ACLs enabled by default
```

### 🏭 Real-Time Production Use Case

**Scenario**: Multi-team development environment — 3 teams need different access to shared directories.

```bash
# Create shared project directory with SGID
mkdir -p /data/projects/myapp
chown root:developers /data/projects/myapp
chmod 2775 /data/projects/myapp   # SGID: new files inherit "developers" group
# All files created here automatically get group "developers" — team sharing works

# Use ACLs for fine-grained access (CI/CD service account needs read-only)
setfacl -m u:jenkins:rx /data/projects/myapp
setfacl -m d:u:jenkins:rx /data/projects/myapp  # default: applies to new files too

# Verify
getfacl /data/projects/myapp
```

### ❌ Common Mistakes

1. **`chmod 777`** — World-writable with execute. Never use in production. It's a security disaster. If you "need" 777, the real fix is correct group assignment or ACLs.
2. **Forgetting the execute bit on directories** — A directory's execute bit means "can enter and access files within." `chmod 644 /mydir` makes the directory unenterable. Directories need at least `711`.
3. **`chmod -R 777 /var/www`** — Recursively setting 777 on a web root allows any process to execute or modify your web files. Use `chmod -R 755` for directories and `chmod -R 644` for files.
4. **Ignoring umask in scripts** — Scripts creating files inherit the running user's umask. Explicitly set permissions with `install -m 0600` or `chmod` after creating sensitive files.

### 🎤 Interview Questions & Answers

**Q: What is the SUID bit and why is it a security concern?**
A: **SUID (Set User ID)** is a special permission bit that, when set on an executable, causes it to run with the **effective user ID of the file's owner** rather than the user who launched it. The classic example is `/usr/bin/passwd` — owned by root with SUID, so any user can run it to change their own password (it writes to `/etc/shadow` which requires root). Security concern: any SUID binary owned by root that has an exploitable vulnerability can be used for **privilege escalation** — an attacker can exploit it to get a root shell. Best practice: run `find / -perm -4000 -type f` regularly to audit SUID binaries, and compare against a known-good baseline. Any unexpected SUID binary is a red flag.

**Q: What is the difference between `chmod 755` and `chmod u=rwx,g=rx,o=rx`?**
A: They're equivalent — just different syntaxes. `755` is octal: 7=rwx (4+2+1) for user, 5=r-x (4+1) for group, 5=r-x for other. `u=rwx,g=rx,o=rx` is symbolic, explicitly setting each category. The octal form is more concise and commonly used in scripts. The symbolic form is more readable for complex cases. A subtlety: the symbolic form with `+` and `-` modifies existing permissions (e.g., `chmod g+w` adds group write without touching other bits). The `=` form sets permissions absolutely (removes bits not listed). The octal form always sets all 9 bits absolutely.

**Q: What is umask and how does it work?**
A: **umask** is a mask that the OS applies to default permissions when creating new files or directories. Default maximum permissions are `0666` for files and `0777` for directories. The umask value is subtracted (bitwise AND with complement). With `umask 0022`: new files get `0666 - 0022 = 0644` (rw-r--r--), new directories get `0777 - 0022 = 0755` (rwxr-xr-x). A stricter umask `0077` gives files `0600` and dirs `0700` — private to the owner. Production servers typically use `0022`. For sensitive environments (handling private keys, secrets), use `0027` so group can read but others cannot.

---

## 12. User & Group Management

### 📖 Concept Explanation

Linux uses a local user database stored in text files: `/etc/passwd` (user info), `/etc/shadow` (password hashes), `/etc/group` (group memberships). Understanding these structures is essential for security auditing and automation.

**`/etc/passwd` format**:
```
username:x:UID:GID:GECOS:home_dir:shell
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
```

**`/etc/shadow` format** (root-only readable):
```
username:$algorithm$salt$hash:lastchange:mindays:maxdays:warndays:inactivedays:expiredate:reserved
```
Password hash types: `$1$`=MD5 (weak), `$5$`=SHA-256, `$6$`=SHA-512 (default on modern systems), `$y$`=yescrypt (Ubuntu 22.04+)

### ⚙️ Key Commands

```bash
# ─── User Management ──────────────────────────────────────────
useradd -m -s /bin/bash -c "Jenkins Service" -G docker,sudo jenkins
# -m: create home dir
# -s: shell (/usr/sbin/nologin for service accounts)
# -c: comment (GECOS)
# -G: supplementary groups

useradd -r -s /usr/sbin/nologin -M myservice
# -r: system account (UID < 1000, no home dir by default)
# -M: do NOT create home dir
# -s /usr/sbin/nologin: cannot log in interactively

passwd jenkins          # set/change password interactively
passwd -l username      # lock account (prepends ! to password hash)
passwd -u username      # unlock account
passwd -e username      # expire password (force change on next login)
usermod -aG docker alice  # add alice to docker group (-a = append, don't overwrite)
usermod -s /bin/bash alice  # change shell
usermod -L alice          # lock user account
userdel -r olduser        # delete user AND home directory (-r removes home)

# ─── Group Management ─────────────────────────────────────────
groupadd developers       # create group
groupadd -g 2000 appteam  # create with specific GID
gpasswd -a alice developers   # add alice to developers group
gpasswd -d alice developers   # remove alice from developers group
gpasswd -M alice,bob,charlie developers  # set exact member list
groups alice              # show alice's groups
id alice                  # uid, gid, and all groups

# ─── User Inspection ──────────────────────────────────────────
id                        # current user: uid, gid, groups
whoami                    # just the username
who                       # who is logged in
w                         # who is logged in + what they're doing + load
last                      # login history from /var/log/wtmp
lastb                     # failed login attempts (bad logins)
lastlog                   # last login for all users
finger username           # detailed user info (if installed)

# ─── /etc/passwd structure ────────────────────────────────────
awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3}' /etc/passwd  # regular users
awk -F: '$7 != "/usr/sbin/nologin" && $7 != "/bin/false" {print $1}' /etc/passwd  # users with valid shells

# ─── su vs sudo ───────────────────────────────────────────────
su alice              # switch to alice (needs alice's password)
su - alice            # switch to alice with login environment
su -                  # switch to root (needs root password)
sudo -u alice command  # run command as alice (needs YOUR sudo rights)
sudo -i               # interactive root shell via sudo
sudo -l               # list your sudo permissions
sudo -l -U alice      # list alice's sudo permissions
```

### 🏭 Real-Time Production Use Case

**Scenario**: Setting up service accounts and groups for a Kubernetes application team.

```bash
# Create application group
groupadd -g 2001 appteam

# Create service account for app (no interactive login)
useradd -r -u 2001 -g appteam -s /usr/sbin/nologin -M -c "MyApp Service Account" myapp

# Create deployment user (CI/CD agent)
useradd -m -u 2002 -g appteam -s /bin/bash -G docker -c "CI/CD Deploy User" deploy

# Set up deploy user with SSH key (no password)
mkdir -p /home/deploy/.ssh
ssh-keygen -t ed25519 -f /tmp/deploy_key -C "deploy@ci" -N ""
install -m 0700 -o deploy -g deploy /home/deploy/.ssh
install -m 0600 -o deploy -g deploy /tmp/deploy_key.pub /home/deploy/.ssh/authorized_keys
passwd -l deploy   # disable password login — SSH key only
```

### ❌ Common Mistakes

1. **Not using `-a` with `usermod -G`** — `usermod -G docker alice` removes alice from ALL groups and adds only docker. The correct command is `usermod -aG docker alice` (`-a` = append).
2. **Deleting a user without `-r`** — `userdel alice` leaves alice's home directory and files. Use `userdel -r alice` or manually clean up `/home/alice` and check for files elsewhere: `find / -user alice 2>/dev/null`.
3. **Service accounts with valid shells** — Service accounts should always use `/usr/sbin/nologin` or `/bin/false` as their shell to prevent interactive login. A service account with `/bin/bash` is a security risk.
4. **Not running `newgrp` or re-logging after adding to a group** — Adding a user to a group takes effect only in NEW sessions. `usermod -aG docker alice` won't take effect for alice's current session. She needs to log out and back in, or run `newgrp docker`.

### 🎤 Interview Questions & Answers

**Q: What's the difference between `su` and `sudo`?**
A: **`su` (switch user)** requires the **target user's password** and starts a new shell as that user. `su -` (with hyphen) simulates a full login — loading the target user's environment. It's an all-or-nothing privilege — you get a full root/user shell. **`sudo` (superuser do)** requires **your own password** and is controlled by the `/etc/sudoers` file. The key advantage of sudo: granular control — you can allow a user to run only specific commands as root (e.g., `alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx`). Sudo logs all commands to `/var/log/auth.log` (syslog), providing an audit trail that `su` doesn't give. Modern production systems prefer sudo over su for accountability and principle of least privilege.

**Q: Explain the `/etc/shadow` file — why does it exist separately from `/etc/passwd`?**
A: Historically, `/etc/passwd` was world-readable (needed to map UIDs to usernames for `ls` etc.). Storing password hashes in a world-readable file allowed offline brute-force attacks. **`/etc/shadow`** was introduced to separate credentials from metadata. `/etc/shadow` is owned by root, readable only by root (and the `shadow` group in some distros). It contains the password hash, password aging information (minimum/maximum age, warning period, expiry date), and account expiry. The `x` in `/etc/passwd`'s password field means "look in /etc/shadow". Modern hashing algorithms in shadow (SHA-512, yescrypt) with salts make dictionary attacks much harder.

---

## 13. Sudo & Privilege Escalation

### 📖 Concept Explanation

**`sudo`** is the gateway to controlled privilege escalation in production. The `/etc/sudoers` file defines exactly who can run what commands as which users.

**`/etc/sudoers` syntax**:
```
WHO     WHERE=(AS_WHO) WHAT
alice   ALL=(ALL) ALL            # alice can run anything as anyone
bob     ALL=(root) /bin/systemctl restart nginx  # bob can only restart nginx
deploy  ALL=(root) NOPASSWD: /usr/bin/docker *   # no password prompt for docker
%ops    ALL=(ALL) NOPASSWD: ALL  # ops group gets passwordless sudo
```

### ⚙️ Key Commands

```bash
# ─── Editing sudoers (NEVER edit directly) ────────────────────
visudo                   # validates syntax before saving (prevents lockout)
visudo -f /etc/sudoers.d/devops  # edit a supplemental sudoers file

# Preferred: drop files in /etc/sudoers.d/ (cleaner, easier to manage with Ansible)
echo "jenkins ALL=(root) NOPASSWD: /usr/bin/systemctl, /usr/bin/docker" \
    | visudo -cf -        # validate without writing (test syntax)

# ─── Using sudo ───────────────────────────────────────────────
sudo command                    # run as root
sudo -u postgres psql           # run as specific user
sudo -l                         # list MY sudo rules
sudo -l -U alice                # list alice's sudo rules (as root)
sudo -i                         # root interactive shell (login shell)
sudo -s                         # root shell (non-login)
sudo -k                         # invalidate sudo timestamp (re-require password next time)

# ─── Sudo logging ─────────────────────────────────────────────
# All sudo usage logged to auth.log / syslog
grep sudo /var/log/auth.log | tail -20
journalctl _COMM=sudo | tail -20  # via journald

# ─── /etc/sudoers.d/ — recommended approach ───────────────────
# Create file: /etc/sudoers.d/90-devops
cat << 'EOF' | sudo tee /etc/sudoers.d/90-devops
# DevOps team — full sudo without password
%devops ALL=(ALL) NOPASSWD: ALL

# Deploy service account — limited to service management
deploy ALL=(root) NOPASSWD: /bin/systemctl start myapp, \
                             /bin/systemctl stop myapp, \
                             /bin/systemctl restart myapp, \
                             /bin/systemctl status myapp

# Ansible — needs broad access, require sudo
ansible ALL=(root) NOPASSWD: ALL
EOF
chmod 440 /etc/sudoers.d/90-devops
```

### 🏭 Real-Time Production Use Case

**Scenario**: CI/CD pipeline needs to restart services but should NOT have full root access.

```bash
# /etc/sudoers.d/99-ci-deploy
jenkins ALL=(root) NOPASSWD: /bin/systemctl restart nginx, \
                              /bin/systemctl restart myapp, \
                              /usr/bin/docker pull *,       \
                              /usr/bin/docker run --rm *

# In Jenkinsfile:
# sh 'sudo systemctl restart nginx'
# Jenkins can restart nginx but cannot rm -rf / or add users
```

### ❌ Common Mistakes

1. **Editing `/etc/sudoers` without `visudo`** — A syntax error in sudoers can lock you out of `sudo` entirely. `visudo` validates before saving. Always use it.
2. **`NOPASSWD: ALL`** for service accounts with shell access — If the service account is compromised (shell injection in a script), attacker gets free root. Whitelist specific commands only.
3. **`sudo su -` instead of `sudo -i`** — Prefer `sudo -i` or `sudo -s` for root shells. `sudo su -` is double-escalation — less clean audit trail and potentially bypasses some sudo controls.
4. **Not knowing privilege escalation vectors** — Common misconfigurations that enable privilege escalation: SUID on bash, writable scripts called by cron as root, `sudo` on editors (vim, nano — can spawn shells), `sudo python3` (can import subprocess and exec anything).

### 🔧 Troubleshooting Tips

```
Problem: "sudo: unable to resolve host <hostname>"
Symptom: sudo commands work but print a warning
Root Cause: /etc/hostname doesn't match an entry in /etc/hosts
Fix:
  hostname                     # check current hostname
  cat /etc/hostname            # expected value
  grep $(hostname) /etc/hosts  # should find a match
  echo "127.0.1.1 $(hostname)" >> /etc/hosts
```

```
Problem: User can't use sudo — "is not in the sudoers file"
Symptom: "alice is not in the sudoers file. This incident will be reported."
Root Cause: User not in sudoers or relevant group
Fix:
  # From root:
  usermod -aG sudo alice     # Debian/Ubuntu (sudo group)
  usermod -aG wheel alice    # RHEL/CentOS (wheel group)
  # OR add specific entry:
  visudo and add: alice ALL=(ALL) ALL
```

### 🎤 Interview Questions & Answers

**Q: How do you give a user the ability to restart only one specific service using sudo, without a password?**
A: Edit `/etc/sudoers` via `visudo` and add:
```
deploy ALL=(root) NOPASSWD: /bin/systemctl restart nginx
```
This means: user `deploy` on any host (`ALL`) can run `sudo /bin/systemctl restart nginx` as root without a password prompt. For more commands, extend it:
```
deploy ALL=(root) NOPASSWD: /bin/systemctl restart nginx, \
                             /bin/systemctl reload nginx
```
Always use the full binary path in sudoers. Put this in `/etc/sudoers.d/deploy` for cleanliness, and set `chmod 440` on the file.

**Q: What are common Linux privilege escalation techniques that a security-aware DevOps engineer should know?**
A: Key vectors to audit: (1) **SUID binaries** — unexpected SUID executables, especially if they run shells or allow file reads; check with `find / -perm -4000`. (2) **Writable scripts run by cron as root** — `crontab -l` for root cron jobs; if the script is writable by a normal user, attacker injects commands. (3) **Dangerous sudo rules** — `sudo vim` lets you `:!bash`; `sudo python3 -c "import os; os.setuid(0); os.system('/bin/bash')"`. Always avoid `sudo` on interpreters or editors. (4) **World-writable /etc/sudoers.d/ files** — files in this directory must be `440` or they're ignored/exploitable. (5) **PATH hijacking** — if a SUID binary calls `ls` without full path, a malicious `ls` in a writable directory earlier in PATH can be executed as root.

---

# 🔥 Section 1–3 Quick Reference Card

```
┌────────────────────────────────────────────────────────────────────┐
│                  LINUX ESSENTIALS — QUICK REFERENCE                 │
├──────────────────────────┬─────────────────────────────────────────┤
│ TOPIC                    │ KEY COMMAND                              │
├──────────────────────────┼─────────────────────────────────────────┤
│ Distro info              │ cat /etc/os-release                     │
│ Kernel version           │ uname -r                                 │
│ Boot analysis            │ systemd-analyze blame                    │
│ GRUB regenerate          │ update-grub / grub2-mkconfig             │
│ Rebuild initramfs        │ dracut --force / update-initramfs -u    │
│ Disk usage               │ du -sh /* | sort -rh | head             │
│ Find large files         │ find / -type f -size +100M              │
│ Find SUID binaries       │ find / -perm -4000 -type f              │
│ Set permissions          │ chmod 755 / chmod u+x / chmod -R 750    │
│ Change owner             │ chown -R user:group /path               │
│ Add user to group        │ usermod -aG groupname username          │
│ Safe sudo edit           │ visudo                                  │
│ List sudo rights         │ sudo -l                                 │
│ Follow logs              │ tail -F /var/log/syslog                 │
│ Top 10 IPs in log        │ awk '{print $1}' log \| sort \| uniq -c \| sort -rn \| head -10 │
│ Source script            │ source script.sh OR . script.sh         │
│ Script safety            │ set -euo pipefail                       │
│ Pipe to file + screen    │ cmd \| tee output.log                   │
│ Run immune to hangup     │ nohup cmd & disown                      │
└──────────────────────────┴─────────────────────────────────────────┘
```

---

*End of Sections 1–3 | Next: Sections 4–6 cover Process Management, Package Management, and Networking*
