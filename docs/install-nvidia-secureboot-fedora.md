# Install NVIDIA Drivers on Fedora + Secure Boot

```
╔══════════════════════════════════════════════════════════════════════════╗
║   NVIDIA  ×  Fedora   —   RPM Fusion · CUDA Repo · Runfile              ║
║   Secure Boot MOK  ·  akmod  ·  Wayland  ·  PRIME  ·  CUDA             ║
╚══════════════════════════════════════════════════════════════════════════╝
```

![Fedora](https://img.shields.io/badge/Fedora-43%2B-blue?logo=fedora)
![NVIDIA](https://img.shields.io/badge/NVIDIA-Driver-76B900?logo=nvidia)
![Secure Boot](https://img.shields.io/badge/Secure%20Boot-MOK%20Enrolled-success)
![Wayland](https://img.shields.io/badge/Wayland-Supported-informational)

Install NVIDIA drivers on Fedora to unlock full GPU acceleration for gaming,
video editing, Blender rendering, CUDA workloads, and machine learning.
This guide covers all three installation paths end-to-end — including Secure
Boot MOK enrollment, akmod timing, Wayland support, and a full troubleshooting
reference.

---

## Table of Contents

- [Before You Start](#before-you-start)
  - [Detect Your GPU](#detect-your-gpu)
  - [Optimus Hybrid Graphics (Laptops)](#optimus-hybrid-graphics-laptops)
  - [Clean Up Previous Installations](#clean-up-previous-installations)
- [Choose Your Installation Method](#choose-your-installation-method)
- [Method 1 — RPM Fusion (Recommended)](#method-1--rpm-fusion-recommended)
  - [Enable RPM Fusion Repos](#enable-rpm-fusion-repos)
  - [Install akmod-nvidia](#install-akmod-nvidia)
  - [Enable Power Management Services](#enable-power-management-services)
  - [Secure Boot and MOK Enrollment](#secure-boot-and-mok-enrollment)
  - [Wait for Akmod Build to Complete](#wait-for-akmod-build-to-complete)
  - [Nouveau Driver Blacklisting](#nouveau-driver-blacklisting)
  - [Wayland Support](#wayland-support)
  - [Desktop-Only vs CUDA Tools](#desktop-only-vs-cuda-tools)
  - [Proprietary vs Open-Source Modules](#proprietary-vs-open-source-modules)
  - [Legacy Drivers (Older GPUs)](#legacy-drivers-older-gpus)
- [Method 2 — NVIDIA CUDA Repository (Older Fedora)](#method-2--nvidia-cuda-repository-older-fedora)
- [Method 3 — Runfile (Last Resort)](#method-3--runfile-last-resort)
- [Verify Your Installation](#verify-your-installation)
- [Hardware Video Acceleration](#hardware-video-acceleration)
- [Troubleshooting](#troubleshooting)
  - [Driver Not Detected / nvidia-smi Missing](#driver-not-detected--nvidia-smi-missing)
  - [Display and Session Issues](#display-and-session-issues)
  - [Kernel Module and Update Issues](#kernel-module-and-update-issues)
  - [Flatpak Can't See the GPU](#flatpak-cant-see-the-gpu)
  - [External Monitor / HDMI Detection (Optimus)](#external-monitor--hdmi-detection-optimus)
  - [Resolution Detection Failures](#resolution-detection-failures)
  - [GUI Updater Reboot Timing Trap](#gui-updater-reboot-timing-trap)
  - [Wayland Performance Issues](#wayland-performance-issues)
  - [VRR / G-SYNC / FreeSync Flickering](#vrr--g-sync--freesync-flickering)
  - [Chrome, Electron, and Wayland](#chrome-electron-and-wayland)
  - [Black Screen in Media Players](#black-screen-in-media-players)
  - [LUKS-Encrypted Fedora (Runfile)](#luks-encrypted-fedora-runfile)
  - [AMD + NVIDIA Hybrid Laptop Boot Hangs](#amd--nvidia-hybrid-laptop-boot-hangs)
- [Dual GPU Management](#dual-gpu-management)
- [Remove NVIDIA Drivers](#remove-nvidia-drivers)
- [Frequently Asked Questions](#frequently-asked-questions)

---

## Before You Start

### Detect Your GPU

Before installing anything, confirm Fedora sees your NVIDIA hardware:

```bash
lspci | grep -i nvidia
```

**Expected output:**

```
01:00.0 VGA compatible controller: NVIDIA Corporation GA106 [GeForce RTX 3060] (rev a1)
```

If nothing appears, you either don't have an NVIDIA GPU or it isn't being
detected by the system — resolve that before proceeding.

---

### Optimus Hybrid Graphics (Laptops)

If `lspci | grep -i nvidia` shows **both** Intel and NVIDIA GPUs, your laptop
uses NVIDIA Optimus. Modern Fedora handles this automatically through PRIME
offloading after installing RPM Fusion drivers — no extra configuration needed
for most workflows.

After driver installation, verify Optimus is working:

```bash
sudo dnf install glx-utils
glxinfo | grep "OpenGL renderer"
```

To force a specific app onto the NVIDIA GPU:

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxgears
```

---

### Clean Up Previous Installations

> **Skip this section** if you have never installed NVIDIA drivers on this
> system before.

Mixing multiple NVIDIA installation methods causes conflicts. Fully remove the
previous installation before switching methods.

**Remove RPM Fusion / CUDA repo packages:**

> [!WARNING]
> The `nvidia*` wildcard may match unrelated packages. Run
> `dnf list installed 'nvidia*'` first to review what will be removed.

```bash
sudo dnf remove nvidia* --allowerasing
```

If you had the CUDA repo enabled, also remove its files:

```bash
sudo rm /etc/yum.repos.d/cuda-fedora*.repo
sudo rm /etc/pki/rpm-gpg/RPM-GPG-KEY-NVIDIA*
```

**Remove a runfile installation:**

```bash
sudo /usr/bin/nvidia-uninstall
```

**Remove a CUDA toolkit runfile installation** (replace `X.Y` with your
version, e.g. `cuda-12.4`):

```bash
sudo /usr/local/cuda-X.Y/bin/cuda-uninstall
```

---

## Choose Your Installation Method

| Method | Driver Freshness | Effort | Secure Boot | Best For |
|--------|-----------------|--------|-------------|----------|
| **RPM Fusion** ✅ | Current (tested) | Very easy | Automatic (MOK) | Everyone on Fedora 43+ |
| **NVIDIA CUDA Repo** | Bleeding-edge | Easy | Manual | Older Fedora (≤42) with CUDA needs |
| **Runfile** | Latest | Complex | Manual (advanced) | Edge cases — last resort only |

> [!IMPORTANT]
> **Use only one method at a time.** Do not mix RPM Fusion, the CUDA
> repository, and the runfile installer on the same system without fully
> removing the previous method first.
>
> **Fedora 43 users:** NVIDIA publishes no Fedora 43 CUDA repository as of
> February 2026. Use RPM Fusion.

---

## Method 1 — RPM Fusion (Recommended)

RPM Fusion is the official community repository for Fedora. It provides
current NVIDIA drivers with seamless kernel integration, automatic updates via
`akmod`, and built-in Secure Boot support. This is the right choice for the
vast majority of Fedora users.

### Enable RPM Fusion Repos

First, bring your system fully up to date:

```bash
sudo dnf upgrade --refresh
```

Add both the Free and Nonfree RPM Fusion repositories. The `$(rpm -E %fedora)`
expression expands automatically to your Fedora release number:

```bash
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

Verify the repos enabled correctly:

```bash
dnf repo list --all | grep -i rpmfusion
```

**Expected output:**

```
rpmfusion-free      RPM Fusion for Fedora 43 - Free      enabled
rpmfusion-nonfree   RPM Fusion for Fedora 43 - Nonfree   enabled
```

---

### Install akmod-nvidia

```bash
sudo dnf install akmod-nvidia
```

> [!CAUTION]
> **Do NOT reboot immediately.** The `akmod` system builds the NVIDIA kernel
> module in the background after DNF finishes — this takes **2–5 minutes**
> depending on your CPU. Rebooting before the build completes will cause the
> system to fall back to Nouveau, making it look like the install failed.
> See [Wait for Akmod Build to Complete](#wait-for-akmod-build-to-complete)
> below before rebooting.

If you need CUDA tools or `nvidia-smi`, also install:

```bash
sudo dnf install xorg-x11-drv-nvidia-cuda
```

---

### Enable Power Management Services

These systemd services preserve GPU state across suspend, hibernate, and
resume — preventing black screens after waking from sleep:

```bash
sudo systemctl enable nvidia-suspend.service
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-resume.service
```

> [!NOTE]
> Do not reboot yet. If Secure Boot is enabled on your system, complete the
> [MOK enrollment steps](#secure-boot-and-mok-enrollment) below first.

---

### Secure Boot and MOK Enrollment

Check whether Secure Boot is active:

```bash
mokutil --sb-state
```

**If Secure Boot is disabled**, skip this section and proceed to
[Wait for Akmod Build to Complete](#wait-for-akmod-build-to-complete).

**If Secure Boot is enabled**, install the signing tools and generate a local
key:

```bash
sudo dnf install kmodtool akmods mokutil openssl
sudo kmodgenca -a
```

> [!NOTE]
> If `kmodgenca -a` reports an existing key pair, keep it. Only use
> `sudo kmodgenca -a --force` when you intentionally need to regenerate it.

Enroll the generated key with MOK:

```bash
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
```

Set a one-time password when prompted, then reboot. At the blue MOK screen:

1. Choose **Enroll MOK**
2. Confirm the key fingerprint
3. Enter the password you just set
4. Select **Continue** → **Reboot**

> [!TIP]
> The MOK enrollment screen almost always uses **US QWERTY** layout regardless
> of your keyboard locale. Use a simple password you can retype accurately.

After rebooting, the signed NVIDIA module will load under Secure Boot. Verify
with `nvidia-smi` (see [Verify Your Installation](#verify-your-installation)).

---

### Wait for Akmod Build to Complete

After `dnf install akmod-nvidia` completes, the kernel module compiles in the
background. Many users think drivers "break after every kernel update" when
they simply rebooted too early.

#### Method 1 — Check Module Availability (simplest)

Run this repeatedly until it returns a version number:

```bash
modinfo -F version nvidia
```

| Output | Meaning |
|--------|---------|
| `modinfo: ERROR: Module nvidia not found.` | Still building — wait |
| `580.119.02` | Build complete — safe to reboot |

#### Method 2 — Watch the Build Log in Real-Time

```bash
sudo tail -f /var/cache/akmods/nvidia/$(uname -r)-$(uname -m)/last.log
```

Or via the journal:

```bash
sudo journalctl --follow --grep=akmod
```

A successful build ends with:

```
Building modules, stage 2.
  MODPOST 5 modules
CC [M]  /var/lib/dkms/nvidia/580.119.02/build/nvidia-drm.mod.o
  LD [M]  /var/lib/dkms/nvidia/580.119.02/build/nvidia-drm.ko
make[1]: Leaving directory '/usr/src/kernels/6.13.5-200.fc43.x86_64'
DKMS: build completed.
```

Press `Ctrl+C` once you see **DKMS: build completed.**

#### Method 3 — Check the kmod Package

```bash
rpm -qa | grep "kmod-nvidia.*$(uname -r)"
```

**Expected output:**

```
kmod-nvidia-6.13.5-200.fc43.x86_64-580.119.02-1.fc43.x86_64
```

If nothing is returned, the build is still in progress or failed — check
the log in Method 2.

#### What If You Already Rebooted Too Early?

Don't reinstall. Boot into your system normally, then:

```bash
sudo akmods --force
sudo reboot
```

The build will also complete automatically on the next boot (just slower).

---

### Nouveau Driver Blacklisting

RPM Fusion's `akmod-nvidia` automatically blacklists Nouveau. After rebooting,
confirm it's gone:

```bash
lsmod | grep nouveau
```

No output = good. If Nouveau is still present, check whether the kmod package
built successfully:

```bash
rpm -qa | grep kmod-nvidia
```

You should see something like `kmod-nvidia-6.13.5-200.fc43.x86_64-580.119.02-1.fc43.x86_64`.
If empty, wait for the akmod build to finish and check `/var/cache/akmods/`.

---

### Wayland Support

NVIDIA drivers 495+ support Wayland. RPM Fusion enables KMS by default, so
Wayland should just work after reboot.

> [!NOTE]
> **Fedora 43+ with GNOME is Wayland-only** — there is no GNOME Xorg session.
> If you need X11 for legacy drivers or specific workflows, use KDE Plasma,
> XFCE, MATE, or Cinnamon.

If Wayland sessions aren't appearing, verify KMS is active:

```bash
cat /proc/cmdline | grep nvidia-drm
```

If `nvidia-drm.modeset=1` is missing, add it:

```bash
sudo grubby --update-kernel=ALL --args='nvidia-drm.modeset=1'
```

Then reboot.

---

### Desktop-Only vs CUDA Tools

| Package | Provides | Install if... |
|---------|----------|---------------|
| `akmod-nvidia` | Kernel driver, OpenGL, Vulkan | Always |
| `xorg-x11-drv-nvidia-cuda` | `nvidia-smi`, CUDA utilities | You need CUDA or want nvidia-smi |

> [!NOTE]
> `nvidia-smi` is **not** included in `akmod-nvidia`. It comes from
> `xorg-x11-drv-nvidia-cuda`. You can add it at any time without reinstalling
> the base driver.

```bash
sudo dnf install xorg-x11-drv-nvidia-cuda
```

---

### Proprietary vs Open-Source Modules

| Module | Package | Best For |
|--------|---------|----------|
| Proprietary (default) | `akmod-nvidia` | All Kepler+ GPUs, gaming, CUDA |
| Open kernel module | `akmod-nvidia-open` | Turing/RTX 20-series+, advanced only |

> [!WARNING]
> RPM Fusion recommends **staying on `akmod-nvidia`** for most users.
> The open kernel module path requires the `rpmfusion-nonfree-tainted` repo
> and is not the default recommendation.

To switch to the open module (advanced, supported GPUs only):

```bash
sudo dnf install rpmfusion-nonfree-release-tainted
sudo dnf swap akmod-nvidia akmod-nvidia-open
sudo reboot
```

---

### Legacy Drivers (Older GPUs)

> [!WARNING]
> Legacy branches are **Xorg-only** and maintained on a best-effort basis.
> They can break with new kernels. Fedora 43+ ships GNOME as Wayland-only —
> use KDE Plasma, XFCE, MATE, or Cinnamon for X11 access with legacy drivers.

**Kepler — GeForce 600/700 series (470xx branch):**

> Some GTX 700-series cards (GTX 750, 750 Ti, some 745 variants) are
> Maxwell-based and use the **current** driver, not 470xx. Check your GPU
> architecture if unsure.

```bash
sudo dnf upgrade --refresh
sudo dnf install xorg-x11-drv-nvidia-470xx akmod-nvidia-470xx
sudo dnf install xorg-x11-drv-nvidia-470xx-cuda
```

**Fermi — GeForce 400/500 series (390xx branch):**

```bash
sudo dnf upgrade --refresh
sudo dnf install xorg-x11-drv-nvidia-390xx akmod-nvidia-390xx
sudo dnf install xorg-x11-drv-nvidia-390xx-cuda
```

**340xx:** Not available on Fedora 43. Use Nouveau, an older Fedora release,
or upgrade your GPU.

If a legacy branch breaks after a kernel update, fall back to Nouveau:

```bash
sudo grubby --update-kernel=ALL --remove-args='rd.driver.blacklist=nouveau,nova_core modprobe.blacklist=nouveau,nova_core'
sudo dracut --force
sudo reboot
```

---

## Method 2 — NVIDIA CUDA Repository (Older Fedora)

> [!IMPORTANT]
> NVIDIA publishes a CUDA repo for **Fedora 42** but not Fedora 43 or newer.
> Verify current support at [NVIDIA CUDA Downloads](https://developer.nvidia.com/cuda-downloads)
> before proceeding. **Fedora 43+ users should use Method 1 (RPM Fusion).**

Update your system first:

```bash
sudo dnf upgrade --refresh
```

### Import the CUDA Repository

Replace `42` with your Fedora version number if different:

```bash
sudo curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/fedora42/x86_64/cuda-fedora42.repo \
  -o /etc/yum.repos.d/cuda-fedora42.repo
```

### Install Dependencies and Drivers

Install kernel headers and DKMS:

```bash
sudo dnf install kernel-headers kernel-devel dkms
```

Browse available packages:

```bash
dnf search nvidia-driver kmod-nvidia
```

Install with DKMS for automatic kernel module rebuilds:

```bash
sudo dnf install nvidia-driver kmod-nvidia-latest-dkms
```

For the open-source kernel module variant (Turing/RTX 20-series or newer only):

```bash
sudo dnf install nvidia-driver kmod-nvidia-open-dkms
```

### Reboot and Verify

```bash
sudo reboot
```

After rebooting:

```bash
nvidia-smi
```

If `nvidia-smi` returns "command not found":

```bash
sudo dnf install nvidia-driver-cuda
```

---

## Method 3 — Runfile (Last Resort)

> [!CAUTION]
> The runfile method is **not recommended** for most users. Known issues
> include Wayland conflicts, X configuration failures, driver loss after
> kernel updates, and a manual rebuild workflow. Only use this if RPM Fusion
> and the CUDA repo both fail to support your GPU or Fedora version.
>
> If Secure Boot is enabled, the runfile method requires a separate manual
> module-signing workflow or temporarily disabling Secure Boot.

### Download the Driver

Navigate to [NVIDIA Driver Downloads](https://www.nvidia.com/Download/index.aspx),
find the correct driver for your GPU, and download the `.run` file. It will
land in `~/Downloads` by default.

### Disable Nouveau (and NOVA Core)

```bash
printf "blacklist nouveau\noptions nouveau modeset=0\nblacklist nova_core\n" | \
  sudo tee /etc/modprobe.d/blacklist-nouveau.conf > /dev/null
```

This blacklists:
- `nouveau` — prevents the open-source driver from loading
- `options nouveau modeset=0` — disables kernel mode-setting for Nouveau
- `nova_core` — blocks the newer NOVA Core module (harmless if not present)

### Regenerate initramfs and Drop to CLI

```bash
sudo dracut --force
sudo systemctl set-default multi-user.target
sudo reboot
```

> [!NOTE]
> If you miss the `set-default` step, you can switch to CLI mode after
> booting with: `sudo systemctl isolate multi-user.target`

### Compiler Compatibility (Fedora 42+)

NVIDIA drivers 580.xx on Fedora 42+ require an explicit C standard flag:

```bash
export CC="gcc -std=gnu17"
```

Skip this step for 470.xx or 390.xx legacy drivers.

### Run the Installer

```bash
cd ~/Downloads
sudo bash NVIDIA-Linux-x86_64-*
```

Or with an explicit version:

```bash
sudo bash NVIDIA-Linux-x86_64-580.119.02.run
```

**Installer prompts:**

| Prompt | Recommended Answer |
|--------|--------------------|
| 32-bit Compatibility Libraries | Yes (benefits some apps) |
| Register kernel modules with DKMS | Yes (enables auto-rebuild on kernel updates) |
| Run nvidia-xconfig | Yes (updates X config automatically) |

### Kernel Module Configuration (Post-Install)

The installer sets `fbdev=0` during CLI mode installation to avoid framebuffer
conflicts. Before rebooting to graphical mode, switch it back:

```bash
## Check current fbdev setting
cat /etc/modprobe.d/nvidia.conf | grep fbdev

## If fbdev=0 is present, change to fbdev=1 for graphical mode
sudo sed -i 's/fbdev=0/fbdev=1/g' /etc/modprobe.d/nvidia.conf
```

### DKMS Compiler Persistence (Fedora 42+)

To ensure future kernel updates compile correctly, add the compiler flag
permanently to the DKMS config:

```bash
## Find your NVIDIA driver version
ls -la /usr/src/ | grep nvidia

## Example: /usr/src/nvidia-580.119.02/
## Edit the dkms.conf file and update the MAKE line
sudo nano /usr/src/nvidia-580.119.02/dkms.conf

## Find the line starting with MAKE[0]= and modify it to include CC flag
## Change from:
## MAKE[0]="'make' -j16 NV_EXCLUDE_BUILD_MODULES='' KERNEL_UNAME=${kernelver} modules"
## To:
## MAKE[0]="CC='gcc -std=gnu17' 'make' -j16 NV_EXCLUDE_BUILD_MODULES='' KERNEL_UNAME=${kernelver} modules"
```

### Re-enable GUI and Reboot

```bash
sudo systemctl set-default graphical.target
sudo reboot
```

---

## Verify Your Installation

### CLI Verification

```bash
# Driver version in kernel
modinfo -F version nvidia

# Full GPU status
nvidia-smi

# GPU memory and compute capability
nvidia-smi --query-gpu=compute_cap,memory.total --format=csv,noheader

# OpenGL renderer — should say NVIDIA, not Mesa
glxinfo | grep "OpenGL version"
```

**Expected `nvidia-smi` output:**

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.119.02             Driver Version: 580.119.02     CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3060      Off   | 00000000:01:00.0  On   |                  N/A |
| 30%   45C    P8              15W / 170W |     512MiB / 12288MiB  |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```

**Expected OpenGL output:**

```
OpenGL version string: 4.6.0 NVIDIA 580.119.02
```

If it shows Mesa or software rendering, the NVIDIA driver didn't load — verify
the akmod build completed and check `lsmod | grep nvidia`.

### GUI Verification

Launch NVIDIA X Server Settings:

```bash
nvidia-settings
```

Or: **Activities → Show Applications → search "NVIDIA X Server"**

---

## Hardware Video Acceleration

NVIDIA GPUs support hardware-accelerated decoding and encoding via VDPAU and
VAAPI. Install the acceleration libraries:

```bash
sudo dnf install libva-nvidia-driver vdpauinfo libva-utils
```

After rebooting with NVIDIA drivers loaded, verify codec support:

```bash
vdpauinfo
```

The output lists every codec your GPU can hardware-accelerate (H.264, HEVC/H.265,
VP9, AV1, MPEG-2, VC-1) along with maximum resolution per codec. Most modern
players (VLC, MPV, Firefox, Chromium) detect these capabilities automatically.

> [!TIP]
> Firefox users: go to **about:preferences → Performance → "Use hardware
> acceleration when available"** to enable it explicitly.

---

## Troubleshooting

### Driver Not Detected / nvidia-smi Missing

**`nvidia-smi` returns "command not found":**

On RPM Fusion, `nvidia-smi` lives in `xorg-x11-drv-nvidia-cuda` — not in
`akmod-nvidia`:

```bash
sudo dnf install xorg-x11-drv-nvidia-cuda
```

Then verify the akmod build completed:

```bash
sudo journalctl -xe | grep -i nvidia
rpm -qa | grep kmod-nvidia
```

**GPU not found (`nvidia-smi` shows "No devices were found"):**

```bash
lspci | grep -i nvidia         # confirm GPU is present
sudo dmesg | grep -i nvidia    # check for initialization errors
```

---

### Display and Session Issues

**Black screen after login (X server fails to start):**

- *Runfile method:* Boot into recovery mode and run `sudo /usr/bin/nvidia-uninstall`, then switch to RPM Fusion.
- *RPM Fusion method:* Verify MOK enrollment completed if Secure Boot is on.
- *Boot config:* Adjust kernel parameters with `grubby`, then regenerate grub config:

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

**Wayland session hangs or crashes:**

- Fedora 42 and earlier: switch to X11 from the login screen's session selector.
- Fedora 43+ (GNOME Wayland-only): switch to KDE Plasma, XFCE, or another desktop that offers X11.
- Advanced: try the open kernel module — switch back if it doesn't help:

```bash
sudo dnf install rpmfusion-nonfree-release-tainted
sudo dnf swap akmod-nvidia akmod-nvidia-open
```

---

### Kernel Module and Update Issues

**Drivers disappear after kernel updates:**

DKMS-enabled packages (`akmod` on RPM Fusion, `latest-dkms` on CUDA repo)
rebuild automatically. Runfile installs require manual reinstallation after
each kernel update.

**DKMS build fails after kernel update:**

Ensure kernel headers match the running kernel:

```bash
uname -r
sudo dnf install kernel-headers-$(uname -r) kernel-devel-$(uname -r)
```

Then force a rebuild:

```bash
sudo akmods --rebuild
```

---

### Flatpak Can't See the GPU

Flatpak maintains its own copy of NVIDIA userland libraries — they must match
your system driver version exactly.

Check your system driver:

```bash
nvidia-smi
```

Check the Flatpak NVIDIA runtime version:

```bash
flatpak list | grep org.freedesktop.Platform.GL.nvidia
```

If the version numbers don't match (e.g., Flatpak shows `580-105-08` but
`nvidia-smi` shows `580.119.02`):

```bash
flatpak update
```

Then reboot.

> [!IMPORTANT]
> After any `dnf` upgrade that updates NVIDIA drivers, **always run
> `flatpak update` before rebooting.** DNF does not update Flatpak runtimes
> automatically.

---

### External Monitor / HDMI Detection (Optimus)

Optimus laptops may fail to detect external displays at boot, even though
plugging in after boot works fine. This usually points to BIOS security
settings conflicting with driver initialization.

**Verify the driver loaded:**

```bash
nvidia-smi
lspci -nnk | grep -iA3 vga
```

Successful output:

```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA106M [GeForce RTX 3060 Mobile / Max-Q] [10de:2560] (rev a1)
        Subsystem: Dell Device [1028:0b5f]
        Kernel driver in use: nvidia
        Kernel modules: nouveau, nvidia_drm, nvidia
```

If `Kernel driver in use` shows `nouveau` or is missing, force rebuild:

```bash
sudo akmods --rebuild --force
sudo reboot
```

**BIOS settings to check (try disabling one at a time):**

| Setting | Why It Causes Issues |
|---------|---------------------|
| Fast Boot / Quick Boot | Skips hardware init steps NVIDIA needs for display detection |
| TPM | Some implementations interfere with GPU init timing |
| CSM / Legacy Boot | Mixed UEFI/Legacy settings cause display output routing problems |
| Secure Boot | Some BIOS implementations block third-party modules even when signed |

> [!NOTE]
> Dell, Lenovo, and HP gaming laptops frequently ship with aggressive security
> defaults. If disabling Secure Boot alone doesn't resolve HDMI detection,
> systematically disable Fast Boot, TPM, and CSM one at a time.

**Symptoms of init problems:**
- System freezes when screen powers off with HDMI connected
- Shutdown hangs with external monitor connected
- Boot message: `nvidia kernel modules not found, falling back to nouveau`

---

### Resolution Detection Failures

Check for multiple conflicting kmod versions:

```bash
rpm -qa | grep kmod-nvidia
uname -r
```

If you see kmod packages for old kernels you no longer boot:

```bash
sudo dnf remove $(rpm -qa | grep 'kmod-nvidia.*6.12.8')
sudo akmods --force
sudo dracut --force
sudo reboot
```

Replace `6.12.8` with the old kernel version. Only remove kmod packages for
kernels you've permanently moved on from.

**Specific driver version broke resolution (downgrade):**

```bash
sudo dnf downgrade akmod-nvidia xorg-x11-drv-nvidia
sudo akmods --force
sudo reboot
```

Verify version after reboot:

```bash
modinfo -F version nvidia
```

**EDID errors / monitor not reporting capabilities:**

```bash
sudo journalctl -b | grep -i edid
xrandr --verbose
```

Try a different cable (DisplayPort vs HDMI) or test with another monitor to
isolate hardware vs driver issues. Also compare available resolutions between
Wayland and X11 sessions — they can differ.

---

### GUI Updater Reboot Timing Trap

GUI package managers (GNOME Software, KDE Discover) prompt you to reboot
immediately after DNF finishes — but `akmod` hasn't built the module yet.

**Symptoms of premature reboot:**

- Black screen or low resolution after update
- `lsmod | grep nvidia` returns nothing
- `lspci -k | grep -A3 VGA` shows `Kernel driver in use: nouveau`

**Fix — use DNF directly:**

```bash
sudo dnf upgrade --refresh
```

Wait for `modinfo -F version nvidia` to return a version number, then reboot
manually:

```bash
sudo reboot
```

**If you already rebooted too early:**

```bash
sudo akmods --force
sudo reboot
```

---

### Wayland Performance Issues

#### KDE Plasma Stuttering — Disable GSP Firmware

Check if GSP firmware is active:

```bash
nvidia-smi -q | grep "GSP Firmware"
```

If output shows a version number (not `N/A`), GSP is enabled and may cause
frame drops in KDE Plasma:

```bash
sudo grubby --update-kernel=ALL --args='nvidia.NVreg_EnableGpuFirmware=0'
sudo reboot
```

#### Black Screen After Suspend — Preserve Video Memory

Check current state:

```bash
sudo grep "PreserveVideoMemoryAllocations" /proc/driver/nvidia/params
```

If output shows `0`:

```bash
sudo grubby --update-kernel=ALL --args='nvidia.NVreg_PreserveVideoMemoryAllocations=1'
sudo reboot
```

Verify it applied:

```bash
sudo grep "PreserveVideoMemoryAllocations" /proc/driver/nvidia/params
```

Expected: `PreserveVideoMemoryAllocations: 1`

---

### VRR / G-SYNC / FreeSync Flickering

High-refresh-rate monitors with VRR enabled frequently experience flickering
and black screen flashes on Wayland with NVIDIA. This is a known limitation
as of February 2026 — the Wayland VRR stack is still maturing.

**Symptoms:**
- Monitor briefly goes black when switching windows
- Heavy flickering in game menus or loading screens
- Issue disappears completely in X11 session

**Fix 1 — Switch to X11 (most reliable):**

At the login screen, click the gear icon and select:
- `GNOME on Xorg` (Fedora 42 and earlier)
- `Plasma (X11)` (KDE, all versions)

Fedora 43+ GNOME users: switch to KDE Plasma or XFCE for X11 access.

**Fix 2 — Disable VRR in desktop settings (Wayland only):**

*GNOME 47+:* Settings → Displays → your monitor → toggle **Variable Refresh Rate** off

*KDE Plasma 6+:* System Settings → Display and Monitor → uncheck **Enable Adaptive Sync**

**Fix 3 — Open kernel modules (Turing+ GPUs):**

```bash
sudo dnf install rpmfusion-nonfree-release-tainted
sudo dnf swap akmod-nvidia akmod-nvidia-open
sudo reboot
```

Switch back if it doesn't help: `sudo dnf swap akmod-nvidia-open akmod-nvidia`

**Fix 4 — Custom EDID table (advanced):**

```bash
sudo dnf install monitor-edid
sudo monitor-get-edid | edid-decode | grep -A5 "Variable Refresh"
```

If your monitor reports a very low minimum VRR frequency (24–30 Hz), increasing
it to 48 Hz in a custom EDID often resolves flickering. Requires familiarity
with `edid-decode`, hex editors, and kernel parameters.

---

### Chrome, Electron, and Wayland

**Chrome, Chromium, Edge:**

Navigate to `chrome://flags` (or `edge://flags`) → search "Preferred Ozone
platform" → set to **Wayland** or **auto** → restart browser.

**Visual Studio Code and Electron apps:**

```bash
code --enable-features=UseOzonePlatform,WaylandWindowDecorations --ozone-platform-hint=auto
```

To make it permanent, add those flags to `~/.config/code-flags.conf`.

---

### Black Screen in Media Players

Black screens in VLC, Firefox, or other players are almost always missing
codecs — not a driver issue:

```bash
sudo dnf config-manager setopt fedora-cisco-openh264.enabled=1
sudo dnf install openh264 mozilla-openh264 libavcodec-freeworld ffmpeg \
  gstreamer1-plugins-bad-freeworld gstreamer1-plugins-ugly
```

Restart your media player or browser after installation.

---

### LUKS-Encrypted Fedora (Runfile)

If your system uses LUKS encryption, add NVIDIA modules to the initramfs
before rebooting after a runfile installation:

```bash
sudo nano /etc/dracut.conf.d/nvidia.conf
```

Add this line:

```
add_drivers+=" nvidia nvidia_modeset nvidia_uvm nvidia_drm "
```

Save, then rebuild:

```bash
sudo dracut --force
```

Now reboot safely.

---

### AMD + NVIDIA Hybrid Laptop Boot Hangs

If your system freezes before reaching the login screen but `startx` works
from a TTY, the issue is likely Wayland compositor initialization, not the
driver itself.

Boot to a text console (`Ctrl+Alt+F3`) and check:

```bash
lsmod | grep nvidia
nvidia-smi
```

**If the driver loaded but graphical sessions fail:**

**Step 1 — Downgrade to previous stable driver:**

```bash
sudo dnf downgrade akmod-nvidia xorg-x11-drv-nvidia
```

Wait for the akmod rebuild:

```bash
sudo tail -f /var/cache/akmods/nvidia/$(uname -r)-$(uname -m)/last.log
```

Then reboot.

**Step 2 — Rebuild initramfs without host-only mode:**

```bash
sudo dracut --force --no-hostonly --kver $(uname -r)
```

This creates a larger initramfs (~150 MB) including all NVIDIA modules for
early boot.

**Step 3 — Check for system service failures:**

```bash
sudo systemctl --failed
sudo journalctl -b -p err
```

Multiple failed services (auditd, firewalld, akmods) may indicate filesystem
corruption or SELinux context issues rather than a driver problem.

**Step 4 — Check kernel module variant:**

```bash
modinfo -l nvidia
```

`NVIDIA` = proprietary module. `Dual MIT/GPL` = open module. RPM Fusion
auto-selects open modules for compatible GPUs (RTX 20-series+). Open modules
sometimes resolve hybrid GPU init timing issues.

> [!TIP]
> If the system won't boot at all, use a Fedora Live USB, mount your root
> partition, and chroot in. From there you can downgrade drivers, rebuild
> initramfs, check logs with `journalctl --list-boots`, and disable
> problematic services.

---

## Dual GPU Management

Install `switcheroo-control` for GPU switching on hybrid systems:

```bash
sudo dnf install switcheroo-control
switcherooctl
```

Launch an application on the discrete NVIDIA GPU:

```bash
switcherooctl launch -g 1 application-name
```

Or use PRIME offloading environment variables directly:

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia application-name
```

Modern desktop environments (GNOME 47+, KDE Plasma 6+) also provide GPU
preference controls per-application in system settings.

---

## Remove NVIDIA Drivers

### RPM Fusion or CUDA Repo

Review what's installed before removing:

```bash
dnf list installed 'nvidia*'
```

Remove driver packages:

```bash
sudo dnf remove akmod-nvidia akmod-nvidia-open xorg-x11-drv-nvidia\* nvidia-settings --allowerasing
```

Verify removal:

```bash
rpm -qa | grep -E 'akmod-nvidia|xorg-x11-drv-nvidia'
```

No output = clean. If you added the CUDA repo, also remove its files:

```bash
sudo rm /etc/yum.repos.d/cuda-fedora*.repo
sudo rm /etc/pki/rpm-gpg/RPM-GPG-KEY-NVIDIA*
```

### Runfile Installation

```bash
sudo /usr/bin/nvidia-uninstall
```

### CUDA Toolkit Runfile

Replace `X.Y` with your version (e.g. `cuda-12.4`):

```bash
sudo /usr/local/cuda-X.Y/bin/cuda-uninstall
```

---

## Frequently Asked Questions

**Are proprietary NVIDIA drivers in Fedora's default repos?**

No. Fedora's default repositories do not ship the proprietary NVIDIA driver.
Use RPM Fusion for most cases. The NVIDIA CUDA repo is a separate option only
for older Fedora releases when NVIDIA publishes a matching repo.

---

**Why is `nvidia-smi` missing after installing `akmod-nvidia`?**

`nvidia-smi` is provided by `xorg-x11-drv-nvidia-cuda`, not `akmod-nvidia`.
Install it separately:

```bash
sudo dnf install xorg-x11-drv-nvidia-cuda
```

Then wait for the kernel module build to finish before running it.

---

**What causes "NVIDIA kernel module missing. Falling back to nouveau"?**

Usually one of two things:
1. The akmod build didn't finish before you rebooted — run `sudo akmods --force` and reboot again.
2. The module failed to build for the current kernel — check `modinfo -F version nvidia` and the akmod build log.

---

**Can I use Wayland with NVIDIA drivers on Fedora 43?**

Yes. Modern NVIDIA drivers fully support Wayland on Fedora 43, and GNOME on
Fedora 43 is Wayland-only. If you need an X11 session for specific workflows
or legacy drivers, use KDE Plasma, XFCE, or another desktop that offers X11.