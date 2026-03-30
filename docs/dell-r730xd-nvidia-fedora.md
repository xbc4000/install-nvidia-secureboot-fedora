# NVIDIA on Dell PowerEdge R730xd — Fedora + RAID6 + Secure Boot

```
╔══════════════════════════════════════════════════════════════════════════╗
║  Dell PowerEdge R730xd  ·  PERC H730 RAID6  ·  Fedora  ·  Secure Boot  ║
║  NVIDIA GPU  ·  iDRAC Fan Fix  ·  MOK Enrollment  ·  akmod             ║
╚══════════════════════════════════════════════════════════════════════════╝
```

![Platform](https://img.shields.io/badge/Platform-Dell%20PowerEdge%20R730xd-blue)
![RAID](https://img.shields.io/badge/RAID-PERC%20H730%20RAID6-orange)
![Fedora](https://img.shields.io/badge/Fedora-43%2B-blue?logo=fedora)
![NVIDIA](https://img.shields.io/badge/NVIDIA-GPU-76B900?logo=nvidia)
![Secure Boot](https://img.shields.io/badge/Secure%20Boot-MOK%20Enrolled-success)

This guide covers everything specific to running an NVIDIA GPU on a
**Dell PowerEdge R730xd** with **PERC H730 RAID6**, **Fedora Linux**, and
**Secure Boot** enabled. It assumes you've already read the main
[NVIDIA driver install guide](install-nvidia-secureboot-fedora.md) and
focuses on server-specific differences, hardware quirks, and the things
that will bite you if you don't know about them.

> [!WARNING]
> Dell does **not officially support** GPUs on the R730xd. The R730 (no "xd")
> is the officially GPU-supported variant. That said, the R730xd has no hard
> BIOS block — GPUs install and run, but you're unsupported. Community members
> run P2000, T400, A2, and similar low-profile/bus-powered cards successfully.
> Treat this as "works in practice, unsupported by Dell."

---

## Table of Contents

- [Hardware Reality Check](#hardware-reality-check)
  - [GPU Compatibility on R730xd](#gpu-compatibility-on-r730xd)
  - [PCIe Slot Layout](#pcie-slot-layout)
  - [Power Requirements](#power-requirements)
  - [Dual CPU Requirement](#dual-cpu-requirement)
- [BIOS Configuration Before Anything Else](#bios-configuration-before-anything-else)
- [The Fan Noise Problem (and the Fix)](#the-fan-noise-problem-and-the-fix)
- [PERC H730 RAID6 and the OS](#perc-h730-raid6-and-the-os)
- [Installing Fedora on a RAID6 R730xd](#installing-fedora-on-a-raid6-r730xd)
- [Install NVIDIA Drivers — Server-Specific Steps](#install-nvidia-drivers--server-specific-steps)
  - [Kernel Headers on Server Kernels](#kernel-headers-on-server-kernels)
  - [SELinux Considerations](#selinux-considerations)
  - [MOK Enrollment with Secure Boot](#mok-enrollment-with-secure-boot)
  - [Akmod Build on a Server](#akmod-build-on-a-server)
- [Enable nvidia-persistenced](#enable-nvidia-persistenced)
- [NVIDIA Power Management on a Server](#nvidia-power-management-on-a-server)
- [Verify Everything Works](#verify-everything-works)
- [PERC RAID Monitoring Alongside NVIDIA](#perc-raid-monitoring-alongside-nvidia)
- [Troubleshooting R730xd-Specific Issues](#troubleshooting-r730xd-specific-issues)
- [iDRAC Reference Commands](#idrac-reference-commands)

---

## Hardware Reality Check

### GPU Compatibility on R730xd

The R730xd is a **storage-first** 2U server — it sacrifices the GPU riser
configuration that the R730 has in exchange for more drive bays. That's the
tradeoff. However, the system board still has PCIe slots and community members
run GPUs on it regularly.

| GPU Type | Status | Notes |
|----------|--------|-------|
| NVIDIA Quadro P2000 / P620 | Works | Low-profile, bus-powered, no fan issue |
| NVIDIA T400 / T1000 | Works | Low-profile, bus-powered |
| NVIDIA A2 | Works | PCIe-only power, server-grade, no fan alarm |
| NVIDIA RTX 3xxx/4xxx (consumer) | Problematic | No GOP support in older BIOS, fan alarm, unsupported |
| NVIDIA Tesla P100 / V100 | Works with power cable | Requires 8-pin PCIe power from riser 3 |
| NVIDIA A30 / A100 | Not tested on R730xd | Designed for newer platforms |

> [!TIP]
> **Best picks for R730xd:** Low-profile, passively cooled or single-fan,
> bus-powered cards like the **P2000**, **T400**, or **A2** cause the least
> trouble. They draw power entirely from the PCIe bus, require no extra power
> connectors, and keep the iDRAC thermal alarm quiet.

> [!IMPORTANT]
> Consumer RTX cards (3070, 3090, 4090, etc.) are not supported. The R730xd's
> BIOS GOP (Graphics Output Protocol) firmware does not have entries for
> consumer GPU ROMs. This can cause no POST video output. Server/Quadro cards
> have GOP support.

---

### PCIe Slot Layout

The R730xd supports **three riser cards**. The layout matters for GPU placement:

| Riser | Slot | Width | Form Factor | Notes |
|-------|------|-------|-------------|-------|
| Riser 1 | Slots 1–3 | x8 (PCIe 3.0) | Half-height, low-profile | Good for low-profile GPUs |
| Riser 2 | Slot 4 | x8 (PCIe 3.0) | Full-height, full-length | |
| Riser 2 | Slot 5 | x16 (PCIe 3.0) | Full-height, full-length | Primary GPU slot |
| Riser 3 | Slot 6 | x16 (PCIe 3.0) | Full-height, full-length | Includes 8-pin 225W GPU power connector |
| Riser 3 | Slot 7 | x8 (PCIe 3.0) | Full-height, full-length | |

**Slot 6 on Riser 3** is the primary GPU slot — it's the only one with a
dedicated 225W power connector. For bus-powered GPUs (≤75W), any slot works.

> [!NOTE]
> Riser 3 is optional and must be ordered separately or sourced aftermarket
> (Dell part ~800JH). Without it, you're limited to Risers 1 and 2.

---

### Power Requirements

| GPU Power Draw | Connector Needed | R730xd Support |
|---------------|-----------------|----------------|
| ≤75W | PCIe bus only | Any slot, any PSU config |
| 75–150W | 6-pin supplemental | Riser 3 slot 6 or external cable |
| 150–225W | 8-pin supplemental | Riser 3 slot 6 (225W connector) |
| >225W | Dual connectors | Not supported on R730xd |

The R730xd supports up to **dual 1100W PSUs** (2×1100W). Always run dual PSU
with a GPU installed. Single PSU + GPU is asking for instability.

> [!WARNING]
> Third-party power cables for "R730-to-GPU" adapters are frequently
> mislabelled and wrong. Community reports confirm many sellers ship
> incompatible cables. If you need a supplemental power cable for a Tesla
> or high-power Quadro, verify the pinout against the R730xd riser spec
> before plugging anything in. A wrong cable can damage the card or the riser.

---

### Dual CPU Requirement

> [!IMPORTANT]
> PCIe slots on Riser 2 and Riser 3 are connected to **CPU 2's PCIe lanes**.
> If only CPU 1 is installed, those slots are **dead** — no device will be
> detected. You must have both CPUs installed to use slots 4–7.
>
> Riser 1 (slots 1–3) is connected to CPU 1 and works with a single CPU.

Check before you pull the server apart:

```bash
# Verify both CPUs are detected by the OS
lscpu | grep "Socket(s)"
# Should show: Socket(s): 2
```

---

## BIOS Configuration Before Anything Else

Access System Setup via **F2 at POST** or through iDRAC → **Launch Virtual Console → Boot to F2**.

Work through these settings in order:

### 1. Confirm UEFI Boot Mode

**System BIOS → Boot Settings → Boot Mode**

Set to `UEFI`. Do **not** use Legacy/BIOS mode — Secure Boot requires UEFI,
and modern Fedora installs expect it.

### 2. Enable Secure Boot (if using MOK)

**System BIOS → System Security → Secure Boot**

Set to `Enabled`. You'll enroll your MOK key during the driver install.
If it's currently `Disabled` and you want Secure Boot, enable it **before**
installing the OS.

### 3. Enable Memory Mapped I/O Above 4 GB

**System BIOS → Integrated Devices → Memory Mapped I/O above 4 GB**

Set to `Enabled`. This is **required** for modern GPUs with >4GB VRAM.
Without it, the GPU may not be addressable and will either fail to initialize
or be limited in capability.

### 4. Disable Embedded Video Controller

**System BIOS → Integrated Devices → Embedded Video Controller**

Set to `Disabled`. This hands off display output to the GPU. Without disabling
it, you may get no output from the NVIDIA card, or the GPU may not initialize
correctly.

> [!NOTE]
> If you disable the embedded video and get no POST output, the NVIDIA card
> may not have initialized yet. If you're using a consumer GPU without GOP
> firmware, you'll get a black screen. In that case, re-enable the embedded
> video or clear NVRAM via the jumper under the server lid to reset.

### 5. Enable SR-IOV (Optional — for virtualization/vGPU)

**System BIOS → Integrated Devices → SR-IOV Global Enable**

Set to `Enabled` if you plan to use GPU passthrough with KVM/QEMU or
NVIDIA vGPU. Not required for bare-metal GPU use.

### 6. Set System Profile for Performance

**System BIOS → System Profile Settings → System Profile**

Set to `Performance` or `Performance Per Watt (OS)`. The default
`Balanced` profile can throttle PCIe performance. For GPU workloads,
`Performance` ensures consistent PCIe and memory bandwidth.

### 7. Enable IPMI over LAN (for fan control)

**iDRAC Settings → Network → IPMI Settings → Enable IPMI over LAN**

Set to `Enabled`. This is required for the fan speed fix in the next section.

---

## The Fan Noise Problem (and the Fix)

This is the most important R730xd-specific gotcha. When you install a
third-party or non-Dell-validated GPU, iDRAC detects an unknown PCIe device
and immediately throws all fans to **full speed** (~18,000 RPM). This is
loud enough to be genuinely unpleasant and will continue indefinitely until
addressed.

The fix is an OEM IPMI command that tells iDRAC to **stop overriding fans
for unknown PCIe cards**. The server still manages thermals normally via
its own algorithm — this just removes the "unknown card = full blast" rule.

### Install ipmitool

```bash
sudo dnf install ipmitool
```

### Disable the Third-Party PCIe Fan Override

Run from any machine that can reach iDRAC's IP on your network:

```bash
ipmitool -I lanplus \
  -H <IDRAC_IP> \
  -U <IDRAC_USERNAME> \
  -P <IDRAC_PASSWORD> \
  raw 0x30 0xce 0x00 0x16 0x05 0x00 0x00 0x00 0x05 0x00 0x01 0x00 0x00
```

Replace `<IDRAC_IP>`, `<IDRAC_USERNAME>`, and `<IDRAC_PASSWORD>` with your
iDRAC credentials.

### Verify It Worked

```bash
ipmitool -I lanplus \
  -H <IDRAC_IP> \
  -U <IDRAC_USERNAME> \
  -P <IDRAC_PASSWORD> \
  raw 0x30 0xce 0x01 0x16 0x05 0x00 0x00 0x00
```

**Response meaning:**

| Response ends with | State |
|-------------------|-------|
| `... 05 00 01 00 00` | Disabled (third-party override off — fans normal) |
| `... 05 00 00 00 00` | Enabled (default — third-party override on — fans at full) |

### Re-enable if Needed

```bash
ipmitool -I lanplus \
  -H <IDRAC_IP> \
  -U <IDRAC_USERNAME> \
  -P <IDRAC_PASSWORD> \
  raw 0x30 0xce 0x00 0x16 0x05 0x00 0x00 0x00 0x05 0x00 0x00 0x00 0x00
```

> [!NOTE]
> This command applies to **13th generation PowerEdge servers** (R730, R730xd,
> R630, R430, R530, R830, R930). It is **not** the same command used on 14th
> gen servers. On newer iDRAC firmware (≥3.34.34.34 on 14th gen), IPMI fan
> control was removed — but the R730xd uses iDRAC 8, which still supports this.

> [!TIP]
> Put this command in a startup script or systemd service so it runs
> automatically after every iDRAC reset or power cycle.

---

## PERC H730 RAID6 and the OS

The PERC H730 presents your RAID6 virtual disk to the OS as a standard
block device — Fedora sees it as a normal disk (e.g. `/dev/sda`) and
has no knowledge of the underlying RAID. This is **hardware RAID**, not
software RAID.

**What this means for NVIDIA driver installation:**

- The `megaraid_sas` kernel module handles the PERC H730. It ships in the
  mainline Linux kernel and is loaded automatically — no extra driver needed.
- There is **no conflict** between `megaraid_sas` and `akmod-nvidia`. They
  operate completely independently.
- DKMS builds both modules separately — PERC from the in-tree kernel driver,
  NVIDIA from akmod/DKMS.
- RAID6 data is fully safe during driver installation — akmod builds happen
  in userspace and don't touch disk I/O at the driver level.

**Check PERC is loaded and happy before touching NVIDIA:**

```bash
# Confirm megaraid_sas is loaded
lsmod | grep megaraid_sas

# List virtual disks presented to OS
lsblk

# Install perccli for RAID health monitoring
# Download from Dell support: https://www.dell.com/support
sudo rpm -ivh perccli_*.noarch.rpm

# Check RAID status
sudo perccli /c0 show
sudo perccli /c0/v0 show
```

**Expected RAID6 status:**

```
VD LIST :
======
DG/VD TYPE  State Access Consist Cache sCC  Size Name
0/0   RAID6 Optl  RW     Yes     RWTD  -   X.XX TB
```

`Optl` = Optimal. If you see `Dgrd` (Degraded) or `Pdbl` (Partially Degraded),
**do not proceed** with driver installation until the array is healthy.

---

## Installing Fedora on a RAID6 R730xd

> [!NOTE]
> Skip this section if Fedora is already installed and you're just adding
> a GPU driver.

### Boot the Installer

1. Download the Fedora Server ISO (or Workstation if you need a GUI)
2. Write to USB: `sudo dd if=Fedora-*.iso of=/dev/sdX bs=4M status=progress`
3. Boot the R730xd from USB via iDRAC Virtual Media or physical USB
4. At the GRUB menu, ensure **UEFI** mode is being used (check for UEFI entries)

### RAID Disk Visibility

The PERC H730 presents the RAID6 virtual disk as a single block device.
Anaconda (Fedora installer) will see it as a regular disk — just pick it
as your installation target. You do **not** need to configure anything RAID-
related in the OS installer.

### Partitioning for UEFI + Secure Boot

Create at minimum:

| Partition | Type | Size | Mount |
|-----------|------|------|-------|
| EFI System Partition | EFI | 512 MB | `/boot/efi` |
| Boot | ext4 or xfs | 1 GB | `/boot` |
| Root | xfs | Remainder | `/` |

> [!IMPORTANT]
> The EFI System Partition **must** exist on the RAID virtual disk for Secure
> Boot to work. The R730xd UEFI firmware boots from the ESP — MOK enrollment
> writes to it. Without a proper ESP, MOK will fail silently.

### Secure Boot During Install

Leave Secure Boot **enabled** in BIOS during installation. Fedora's installer
(Anaconda) and bootloader (GRUB2 with shim) are Secure Boot signed. The system
will boot normally after install — you enroll the NVIDIA MOK key separately
as part of driver installation.

---

## Install NVIDIA Drivers — Server-Specific Steps

Follow the main [RPM Fusion install guide](install-nvidia-secureboot-fedora.md)
for the core steps. The sections below cover R730xd-specific differences.

### Kernel Headers on Server Kernels

On a server you're more likely to be running headless with a stock kernel,
but always confirm headers match **exactly**:

```bash
# Check running kernel
uname -r

# Install matching headers and devel (required for akmod/DKMS)
sudo dnf install \
  kernel-headers-$(uname -r) \
  kernel-devel-$(uname -r)

# Verify they match
rpm -qa | grep kernel-devel
# Should show exactly your running kernel version
```

> [!CAUTION]
> On servers it's common to defer kernel updates for stability. If your running
> kernel is older than the latest available, `dnf install kernel-headers`
> (without specifying version) will pull the **latest** headers, which won't
> match your running kernel. Always specify `$(uname -r)` explicitly.

---

### SELinux Considerations

Fedora runs SELinux in **Enforcing** mode by default. The akmod build process
and NVIDIA driver work fine with SELinux enforcing — RPM Fusion packages are
properly labelled. However, watch for these:

**After driver install, if nvidia-smi fails with permission errors:**

```bash
# Check for SELinux denials
sudo ausearch -m avc -ts recent | grep nvidia

# If denials exist, check the context on nvidia device files
ls -laZ /dev/nvidia*

# Restore correct SELinux context if wrong
sudo restorecon -Rv /dev/nvidia*
```

**If akmod fails to build and you see SELinux errors in journald:**

```bash
sudo journalctl -b | grep -i 'avc.*denied.*akmod'
```

In rare cases, a SELinux denial during akmod's kernel module build can cause
a silent failure. If `modinfo -F version nvidia` never returns a version after
10+ minutes, check SELinux before assuming an akmod problem.

---

### MOK Enrollment with Secure Boot

The process is identical to the main guide, but on a server you'll likely
do MOK enrollment via **iDRAC Virtual Console** rather than a physical monitor.

```bash
# Check Secure Boot state
mokutil --sb-state

# Install tools and generate key
sudo dnf install kmodtool akmods mokutil openssl
sudo kmodgenca -a

# Enroll the key
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
# Set a simple password — you'll type it at the MOK screen
```

**Reboot and complete enrollment via iDRAC Virtual Console:**

1. Open iDRAC web UI → **Virtual Console**
2. Reboot the server
3. Watch for the blue MOK Manager screen during boot
4. Select **Enroll MOK** → **Continue** → **Yes** → enter your password → **Reboot**

> [!TIP]
> The MOK screen has a short timeout (~90 seconds). If you miss it, the
> server boots normally without enrolling the key and the NVIDIA module won't
> load. Just reboot and try again — `mokutil --import` only needs to be run
> once; the pending enrollment persists across reboots until enrolled or cleared.

> [!NOTE]
> iDRAC Virtual Console may have slight keyboard latency. Type your MOK
> password slowly and deliberately. Use a short, simple password with only
> standard alphanumeric characters to avoid layout issues.

---

### Akmod Build on a Server

On a headless server, you won't have a desktop to watch. Monitor the build
from an SSH session:

```bash
# Watch build log in real-time
sudo tail -f /var/cache/akmods/nvidia/$(uname -r)-$(uname -m)/last.log

# Or watch via journal
sudo journalctl --follow --grep=akmod

# Poll until ready (run this in a loop)
watch -n 10 'modinfo -F version nvidia'
```

On a server-class CPU (Xeon E5 v3/v4), the build typically takes **2–4 minutes**.
Don't reboot until `modinfo -F version nvidia` returns a version number.

```bash
# Confirm before rebooting
modinfo -F version nvidia         # version number = safe to reboot
rpm -qa | grep "kmod-nvidia.*$(uname -r)"  # kmod package = safe to reboot
```

---

## Enable nvidia-persistenced

On a server, `nvidia-persistenced` is important. Without it, the NVIDIA driver
unloads GPU state when no process is using the GPU — this adds significant
latency to the first CUDA operation after idle and can cause issues with
monitoring tools.

```bash
# Enable and start persistence daemon
sudo systemctl enable nvidia-persistenced
sudo systemctl start nvidia-persistenced

# Verify it's running
systemctl status nvidia-persistenced
```

**Set persistence mode via nvidia-smi as well:**

```bash
sudo nvidia-smi -pm 1
# Output: Persistence mode has been set to "On" for GPU #0
```

> [!NOTE]
> Persistence mode and `nvidia-persistenced` serve slightly different purposes.
> `nvidia-persistenced` keeps the driver loaded at the OS level.
> `nvidia-smi -pm 1` keeps the GPU in a high-power ready state.
> On a server doing compute work, enable both.

---

## NVIDIA Power Management on a Server

Server-grade cards (A2, T4, P100, V100) have configurable power limits and
ECC memory. Consumer cards in a server context skip most of this.

### Check GPU Status

```bash
# Full GPU info — power draw, temp, utilization, ECC state
nvidia-smi

# Detailed query
nvidia-smi -q

# Power limits
nvidia-smi -q -d POWER

# ECC error counts (server cards only)
nvidia-smi -q -d ECC
```

### Set Power Limit (Server Cards)

```bash
# Check current and maximum power limits
nvidia-smi -q -d POWER | grep -E "Power Limit|Max Power"

# Set power limit (example: 70W for a T400)
sudo nvidia-smi -pl 70
```

### ECC Memory (Server Cards)

Some NVIDIA server cards (A2, T4, A30) support ECC memory. Check and configure:

```bash
# Check ECC state
nvidia-smi -q -d ECC | grep "ECC Mode"

# Enable ECC (requires reboot)
sudo nvidia-smi --ecc-config=1 -i 0

# Disable ECC (frees ~6% more VRAM, requires reboot)
sudo nvidia-smi --ecc-config=0 -i 0
```

### Enable Accounting Mode

Accounting mode tracks GPU utilization per-process — useful on a server:

```bash
sudo nvidia-smi -am 1
# Check
nvidia-smi --query-accounted-apps=gpu_name,time,gpu_util --format=csv
```

---

## Verify Everything Works

```bash
# GPU detected and driver loaded
nvidia-smi

# Kernel module loaded
lsmod | grep nvidia

# PERC RAID still healthy
sudo perccli /c0 show

# Nouveau is dead
lsmod | grep nouveau    # no output = good

# Secure Boot is still active
mokutil --sb-state      # SecureBoot enabled

# MOK key is enrolled
mokutil --list-enrolled | grep -i nvidia   # or your key's subject

# SELinux still enforcing
getenforce               # Enforcing

# Both CPUs online (PCIe slots accessible)
lscpu | grep "Socket(s)"   # Socket(s): 2

# iDRAC reachable
ping <IDRAC_IP>
```

**Full nvidia-smi output on a working system:**

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.xx.xx              Driver Version: 580.xx.xx    CUDA Version: 12.x       |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|=========================================+========================+======================|
|   0  NVIDIA Quadro P2000          On    | 00000000:03:00.0  Off  |                  N/A |
| 41%   42C    P8               5W /  75W |      0MiB /  5120MiB  |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```

---

## PERC RAID Monitoring Alongside NVIDIA

Keep an eye on the RAID array health, especially after kernel updates that
trigger `akmod` rebuilds (disk I/O heavy):

```bash
# Install perccli if not already done
sudo rpm -ivh perccli_*.noarch.rpm

# Virtual disk health
sudo perccli /c0/vall show

# Physical disk health
sudo perccli /c0/eall/sall show

# Controller events log
sudo perccli /c0 show events

# Watch rebuild progress (if a disk is rebuilding)
watch -n 30 'sudo perccli /c0/vall show | grep -E "State|Progress"'
```

**Set up a basic health check alias:**

```bash
echo "alias raidcheck='sudo perccli /c0 show && sudo perccli /c0/eall/sall show | grep -E \"State|Intf|Med\"'" >> ~/.bashrc
source ~/.bashrc
raidcheck
```

---

## Troubleshooting R730xd-Specific Issues

### GPU Not Detected at Boot

```bash
lspci | grep -i nvidia
```

If nothing appears:

1. **Check slot:** Are you in Riser 2/3 with only one CPU? Slots 4–7 need both CPUs.
2. **Check BIOS:** Is "Memory Mapped I/O above 4 GB" enabled?
3. **Check embedded video:** Is it disabled? A conflicting embedded controller can prevent GPU init.
4. **Reseat the card:** PCIe seating issues are common with heavy GPUs in 2U servers.
5. **Check power:** Bus-powered GPU in a slot with insufficient power delivery? Try a different slot.

### No Video Output from GPU

The R730xd embedded video controller takes priority at POST. If you installed
the GPU and still see output from the embedded controller:

```bash
# Disable embedded video in BIOS:
# F2 → System Setup → Integrated Devices → Embedded Video Controller → Disabled
```

If you get a black screen after disabling embedded video:
- The GPU may lack GOP firmware (common with consumer cards)
- Clear NVRAM: locate the NVRAM clear jumper on the motherboard (see R730xd
  Owner's Manual for jumper location), clear it to reset BIOS video settings

### Fan Still at Full Speed After IPMI Command

- Verify IPMI over LAN is enabled in iDRAC settings
- Confirm the command returned `05 00 01 00 00` (disabled state)
- Check iDRAC firmware version — very old iDRAC 8 firmware may behave differently
- Try updating iDRAC firmware first, then re-run the command

```bash
# Check iDRAC firmware version from OS
sudo dmidecode -t 38 | grep Version
```

### NVIDIA Driver Loads but GPU Shows Error State

```bash
sudo dmesg | grep -i nvidia | grep -i error
nvidia-smi   # look for "ERR!" in the output
```

Common causes on R730xd:
- **Power delivery issue:** GPU drawing more than the slot/cable can supply
- **Thermal issue:** Server fans at full but GPU still overheating (verify GPU has adequate airflow — server airflow is front-to-back)
- **PCIe power state conflict:** Try disabling C-states in BIOS

### akmod Builds But Driver Doesn't Load After Reboot

```bash
# Check if Secure Boot rejected the module
sudo dmesg | grep -i "PKCS#7\|signature\|required key"
```

If you see signature/key errors, MOK enrollment didn't complete. Re-run:

```bash
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
sudo reboot
# Complete enrollment at the MOK blue screen via iDRAC Virtual Console
```

### PERC H730 Shows Degraded After Driver Install

The NVIDIA akmod build is CPU and I/O intensive. In rare cases on degraded
arrays, this can expose pre-existing disk issues. Check:

```bash
sudo perccli /c0/vall show
sudo perccli /c0/eall/sall show | grep -v Onln
# Any non-"Onln" (Online) disks are a problem
```

If a disk shows `Rbld` (Rebuilding) after the install — that's fine, it
started rebuilding. If it shows `Failgd` or `Msng`, you have a disk problem
that predates the GPU install.

---

## iDRAC Reference Commands

Useful iDRAC/IPMI commands for managing the server alongside the NVIDIA GPU:

```bash
# Disable third-party PCIe fan override (quieten fans)
ipmitool -I lanplus -H <IP> -U <USER> -P <PASS> \
  raw 0x30 0xce 0x00 0x16 0x05 0x00 0x00 0x00 0x05 0x00 0x01 0x00 0x00

# Check fan override status
ipmitool -I lanplus -H <IP> -U <USER> -P <PASS> \
  raw 0x30 0xce 0x01 0x16 0x05 0x00 0x00 0x00

# Re-enable third-party fan override (restore Dell default)
ipmitool -I lanplus -H <IP> -U <USER> -P <PASS> \
  raw 0x30 0xce 0x00 0x16 0x05 0x00 0x00 0x00 0x05 0x00 0x00 0x00 0x00

# Read all sensor data (temps, fan speeds, voltages)
ipmitool -I lanplus -H <IP> -U <USER> -P <PASS> sdr list

# Read fan speeds only
ipmitool -I lanplus -H <IP> -U <USER> -P <PASS> sdr type Fan

# Read temperatures
ipmitool -I lanplus -H <IP> -U <USER> -P <PASS> sdr type Temperature

# Power cycle the server
ipmitool -I lanplus -H <IP> -U <USER> -P <PASS> chassis power cycle

# Graceful shutdown
ipmitool -I lanplus -H <IP> -U <USER> -P <PASS> chassis power soft

# Check system event log
ipmitool -I lanplus -H <IP> -U <USER> -P <PASS> sel list
```

> [!NOTE]
> These IPMI commands apply to **iDRAC 8** on 13th generation PowerEdge
> servers (R730xd). The R730xd uses iDRAC 8. The `0x30 0xce` OEM command
> space is specific to Dell 13G servers and will not work on other vendors
> or newer Dell generations.

---

## Summary — R730xd NVIDIA Checklist

Before you reboot into production:

- [ ] Both CPUs installed (required for Riser 2/3 PCIe slots)
- [ ] BIOS: Boot Mode = UEFI
- [ ] BIOS: Secure Boot = Enabled
- [ ] BIOS: Memory Mapped I/O above 4 GB = Enabled
- [ ] BIOS: Embedded Video Controller = Disabled
- [ ] BIOS: SR-IOV = Enabled (if using passthrough)
- [ ] BIOS: System Profile = Performance
- [ ] BIOS: IPMI over LAN = Enabled
- [ ] GPU seated in correct slot (Slot 5 or 6 preferred)
- [ ] Power cable verified if GPU needs supplemental power
- [ ] PERC RAID6 showing Optimal before starting driver install
- [ ] `sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda`
- [ ] Power management services enabled (`nvidia-suspend`, `hibernate`, `resume`)
- [ ] MOK key generated and enrolled via iDRAC Virtual Console
- [ ] Waited for `modinfo -F version nvidia` to return version number
- [ ] Rebooted and confirmed `nvidia-smi` shows GPU
- [ ] `nvidia-persistenced` enabled and running
- [ ] IPMI fan override disabled (quiet fans)
- [ ] RAID still Optimal after reboot (`sudo perccli /c0 show`)
- [ ] SELinux still Enforcing (`getenforce`)

---

*See also: [Main NVIDIA Fedora Install Guide](install-nvidia-secureboot-fedora.md)*
