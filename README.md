# NVIDIA Drivers on Fedora + Secure Boot

```
╔══════════════════════════════════════════════════════════════════════╗
║  Fedora  ·  RPM Fusion  ·  akmod  ·  Secure Boot MOK  ·  Wayland    ║
║  Three install methods  ·  Full troubleshooting  ·  CUDA ready       ║
╚══════════════════════════════════════════════════════════════════════╝
```

Getting NVIDIA drivers working on Fedora is one of those tasks that
looks simple, bites you in the face, and then suddenly clicks. This
guide covers all three installation methods end-to-end — including
Secure Boot MOK enrollment, akmod timing traps, Wayland quirks, and
a full troubleshooting reference so you don't spend three hours
staring at a black screen wondering what went wrong.

---

## What's In Here

| Doc | What It Covers |
|-----|---------------|
| [install-nvidia-secureboot-fedora.md](docs/install-nvidia-secureboot-fedora.md) | Full install guide — all three methods, Secure Boot, verification, hardware acceleration, troubleshooting, FAQ |

---

## Pick Your Method

| Method | Best For | Secure Boot | Effort |
|--------|----------|-------------|--------|
| **RPM Fusion** (recommended) | Everyone on Fedora 43+ | Automatic (MOK) | Low |
| **NVIDIA CUDA Repo** | Older Fedora (≤42) with CUDA needs | Manual | Medium |
| **Runfile** | Edge cases only — last resort | Manual (advanced) | High |

**TL;DR:** Use RPM Fusion. NVIDIA has no Fedora 43 CUDA repo. The runfile
method will make you sad. Start here:

```bash
# Check you actually have an NVIDIA GPU first
lspci | grep -i nvidia

# Update before touching drivers
sudo dnf upgrade --refresh

# Add RPM Fusion (free + nonfree)
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

# Install the driver — akmod rebuilds automatically on kernel updates
sudo dnf install akmod-nvidia

# DO NOT REBOOT YET — wait for akmod to finish (2-5 min)
# Watch for it:
modinfo -F version nvidia   # keep running until you see a version number
```

---

## The #1 Trap Everyone Falls Into

```
You install the driver.
DNF says "Complete!"
You reboot immediately.
Black screen. Nouveau is back. You panic.
```

**What actually happened:** `akmod` builds the kernel module *after* DNF
finishes. It takes 2–5 minutes. Reboot before it's done and the module
doesn't exist yet.

**The fix:** After DNF completes, run `modinfo -F version nvidia` in a
loop. When it returns a version number (e.g. `580.119.02`) instead of
an error, *then* reboot. The guide has three verification methods.

---

## Secure Boot (MOK Enrollment)

If Secure Boot is on, the NVIDIA kernel module needs to be signed with
a key your firmware trusts. RPM Fusion handles this with `akmods` + MOK.

```bash
# Check if Secure Boot is enabled
mokutil --sb-state

# If enabled — install tools and generate signing key
sudo dnf install kmodtool akmods mokutil openssl
sudo kmodgenca -a

# Enroll the key (sets a one-time password for the MOK screen)
sudo mokutil --import /etc/pki/akmods/certs/public_key.der

# Reboot → blue MOK enrollment screen appears
# Choose "Enroll MOK" → confirm key → enter your password → reboot
```

> **MOK screen tip:** It almost always uses US QWERTY layout regardless
> of your keyboard locale. Use a simple password you can retype.

Full step-by-step in the [install guide](docs/install-nvidia-secureboot-fedora.md#secure-boot-and-mok-enrollment-on-rpm-fusion).

---

## Verify It Worked

```bash
# Driver version loaded in kernel
modinfo -F version nvidia

# GPU status, driver version, CUDA version, memory usage
nvidia-smi

# OpenGL renderer — should say NVIDIA, not Mesa
glxinfo | grep "OpenGL version"

# Nouveau should be dead and gone
lsmod | grep nouveau   # no output = good
```

> `nvidia-smi` is in `xorg-x11-drv-nvidia-cuda`, not `akmod-nvidia`.
> If it's missing: `sudo dnf install xorg-x11-drv-nvidia-cuda`

---

## What's Covered in the Full Guide

- **Pre-install:** GPU detection, Optimus hybrid graphics, cleaning up
  previous installations
- **Method 1 — RPM Fusion:** akmod install, power management services,
  Secure Boot MOK enrollment, Nouveau blacklisting, Wayland KMS, legacy
  driver branches (470xx, 390xx)
- **Method 2 — NVIDIA CUDA Repo:** Fedora 42 and older, DKMS setup,
  open vs proprietary module choice
- **Method 3 — Runfile:** Nouveau blacklisting, compiler flags for
  Fedora 42+ (`gcc -std=gnu17`), DKMS persistence, fbdev parameter,
  re-enabling the GUI
- **Hardware video acceleration:** VDPAU + VAAPI, `libva-nvidia-driver`,
  codec verification
- **Wayland:** KMS confirmation, GSP firmware tweak for KDE Plasma,
  PreserveVideoMemoryAllocations for suspend/resume
- **VRR / G-SYNC flickering:** Root cause, X11 workaround, disabling
  VRR per DE, custom EDID approach
- **Full troubleshooting section:**
  - Driver not detected / nvidia-smi missing
  - Nouveau still present after install
  - X server black screen after login
  - Drivers not surviving kernel updates
  - DKMS build failures
  - Flatpak apps can't see GPU after driver update
  - External monitor / HDMI detection on Optimus laptops
  - Resolution detection failures and conflicting kmod versions
  - GUI package manager reboot timing trap (GNOME Software / KDE Discover)
  - LUKS-encrypted Fedora initramfs setup
  - AMD+NVIDIA hybrid laptop boot hangs
  - Chrome/Electron Wayland flags
  - Black screen in VLC/Firefox (missing codecs)
- **Dual GPU management:** `switcheroo-control`, PRIME offloading
- **Removal:** Clean uninstall for all three methods
- **FAQ:** 4 common questions answered directly

---

## Quick Reference

### Enable Power Management (do this before first reboot)

```bash
sudo systemctl enable nvidia-suspend.service
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-resume.service
```

### Hardware Video Acceleration

```bash
sudo dnf install libva-nvidia-driver vdpauinfo libva-utils
vdpauinfo   # verify codecs after reboot
```

### Fix Flatpak GPU Detection After Driver Update

```bash
flatpak update   # always run this after dnf upgrades NVIDIA packages
```

### Force App to Use NVIDIA GPU (Optimus)

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia your-app
# or
switcherooctl launch -g 1 your-app
```

### Wayland: Fix KDE Plasma GSP Stuttering

```bash
sudo grubby --update-kernel=ALL --args='nvidia.NVreg_EnableGpuFirmware=0'
sudo reboot
```

### Wayland: Fix Black Screen After Suspend

```bash
sudo grubby --update-kernel=ALL --args='nvidia.NVreg_PreserveVideoMemoryAllocations=1'
sudo reboot
```

### Nuclear Option — Full Driver Removal

```bash
dnf list installed 'nvidia*'
sudo dnf remove akmod-nvidia akmod-nvidia-open xorg-x11-drv-nvidia\* nvidia-settings --allowerasing
```

---

## Notes

- Fedora 43 ships GNOME as **Wayland-only** — no GNOME Xorg session.
  If you need X11, use KDE Plasma, XFCE, MATE, or Cinnamon.
- NVIDIA has **no Fedora 43 CUDA repo** as of February 2026 — use RPM Fusion.
- Legacy drivers (470xx, 390xx) are Xorg-only and best-effort on new kernels.
- The runfile method requires manual reinstall after every kernel update
  unless you configure DKMS — and even then, it's more fragile than akmod.
- Dell/Lenovo/HP gaming laptops often ship with aggressive BIOS security
  defaults that break HDMI detection — Fast Boot and TPM are common culprits.
