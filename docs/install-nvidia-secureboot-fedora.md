> **Source:** This document is based on the original guide by LinuxCapable:
> [How to Install NVIDIA Drivers on Fedora Linux](https://linuxcapable.com/how-to-install-nvidia-drivers-on-fedora-linux/)
> — visit the original for screenshots and visual walkthroughs.

---

Install NVIDIA drivers on Fedora to unlock full GPU acceleration for gaming, video editing, Blender rendering, CUDA workloads, and machine learning. Fedora supports multiple installation paths with different trade-offs between convenience, driver freshness, and system integration.

RPM Fusion is the recommended installation method for most Fedora users because it provides current drivers with automatic updates, Fedora integration, and Secure Boot support through akmods. Fedora’s default repositories do not ship the proprietary NVIDIA driver. As of February 23, 2026, NVIDIA does not publish a Fedora 43 CUDA repository, so Fedora 43 users should use RPM Fusion. Fedora 43 and newer also ship GNOME as Wayland-only; if you need an X11 session for legacy drivers or specific workflows, use a desktop that still offers X11 (for example KDE Plasma, XFCE, MATE, or Cinnamon). This guide covers RPM Fusion first, then the NVIDIA CUDA repository for older Fedora releases with supported CUDA repos, and finally the runfile method for edge cases.
On this page
hide
Install NVIDIA Drivers on Fedora
Choose Your NVIDIA Driver Installation Method on Fedora
Verify NVIDIA Drivers on Fedora
Enable Hardware Video Acceleration with NVIDIA Drivers on Fedora
Troubleshoot Common NVIDIA Driver Issues on Fedora
Remove NVIDIA Drivers on Fedora
Frequently Asked Questions
Conclusion
Install NVIDIA Drivers on Fedora

Before installing drivers, verify that your system has an NVIDIA GPU installed. Open a terminal and run:

lspci | grep -i nvidia

Example output confirming Fedora detects an NVIDIA GPU:

01:00.0 VGA compatible controller: NVIDIA Corporation GA106 [GeForce RTX 3060] (rev a1)

This command displays all NVIDIA graphics devices connected to your system. The output shows your GPU model (for example, “NVIDIA Corporation GF119” or “NVIDIA Corporation GA106”), which helps you verify driver compatibility and ensure you have an NVIDIA GPU before proceeding with installation.
NVIDIA Optimus Hybrid Graphics on Fedora Laptops

If your lspci | grep -i nvidia output shows both Intel and NVIDIA GPUs, your laptop uses NVIDIA Optimus technology for power management, which switches between integrated Intel graphics (for battery life) and discrete NVIDIA graphics (for performance) automatically. Modern Fedora handles Optimus through PRIME offloading when you install RPM Fusion drivers; no additional configuration is needed for most workflows.

After installing NVIDIA drivers through RPM Fusion (covered below), your system automatically enables PRIME offloading. To verify Optimus configuration works correctly after driver installation, install the OpenGL utilities package and check available GPUs:

sudo dnf install glx-utils
glxinfo | grep "OpenGL renderer"

You can force specific applications to use the NVIDIA GPU by prefixing commands with environment variables:

__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxgears

This example runs glxgears on the NVIDIA GPU instead of integrated Intel graphics. Use the same environment variables with your applications to force the discrete GPU when needed.
Clean Up Previous NVIDIA Installations Before Reinstalling on Fedora

If you previously installed NVIDIA drivers on your Fedora system and want to try a different installation method, proceed with caution. Combining multiple NVIDIA repositories can cause driver conflicts. Revert to the default drivers before switching to a new installation method.

Skip this section if you have not installed NVIDIA drivers on this Fedora system before.
Remove NVIDIA Drivers with DNF

    The nvidia* wildcard may match unrelated packages. Run dnf list installed 'nvidia*' first to review what will be removed. Consider using a more specific package list like akmod-nvidia xorg-x11-drv-nvidia\* to avoid accidental removal of packages you want to keep.

To remove traces of previous NVIDIA installations, run the following command:

sudo dnf remove nvidia* --allowerasing

Remember to delete the repository files depending on the method you used to install them. For RPM Fusion, you can leave the repository enabled for future use. If you installed via the NVIDIA CUDA repository and want to remove it:

sudo rm /etc/yum.repos.d/cuda-fedora*.repo
sudo rm /etc/pki/rpm-gpg/RPM-GPG-KEY-NVIDIA*

Remove Runfile NVIDIA Installation

If you opted for the “.run” file method to install NVIDIA drivers, you’ll need a distinct removal process.

To uninstall the runfile type of installation, execute:

sudo /usr/bin/nvidia-uninstall

Remove CUDA Toolkit Runfile Installation

If you installed the CUDA toolkit using the runfile approach, remove it with the built-in uninstaller. Replace X.Y with your installed version (for example, cuda-12.4):

sudo /usr/local/cuda-X.Y/bin/cuda-uninstall

Choose Your NVIDIA Driver Installation Method on Fedora

Fedora offers three installation paths for NVIDIA drivers. The table below compares each method’s trade-offs:
Installation Method	Driver Recency	Ease of Use	Secure Boot	Best For	Channel
RPM Fusion	Current (tested)	Very Easy	✅ Yes (MOK)	Most users on Fedora 43+ and users who want stable, integrated driver updates	RPM Fusion NVIDIA Howto
NVIDIA CUDA Repository	Bleeding-edge	Easy	⚠️ Manual	Older Fedora releases with a matching NVIDIA CUDA repo (for example Fedora 42)	NVIDIA CUDA Downloads
Runfile Method	Latest	Complex	⚠️ Manual (advanced)	Edge cases; last resort only	NVIDIA Driver Downloads

Recommendation: Start with RPM Fusion unless you have a specific reason to use another method. RPM Fusion provides automatic kernel module rebuilds (akmod), Secure Boot integration, and timely driver updates through Fedora’s normal update workflow. Fedora 43 users should use RPM Fusion because NVIDIA does not currently publish a Fedora 43 CUDA repository. The runfile method is best reserved for unsupported edge cases.

    Use only one NVIDIA driver installation method at a time. Do not mix RPM Fusion, the NVIDIA CUDA repository, and the runfile installer on the same Fedora installation unless you fully remove the previous method first.

Method 1: Install NVIDIA Drivers on Fedora via RPM Fusion (Recommended)

RPM Fusion is the official community repository for Fedora and provides current NVIDIA drivers with seamless kernel integration, automatic updates, and Secure Boot support. This is the recommended installation method for the vast majority of Fedora users.

Before installing, ensure your system is updated to prevent potential conflicts between graphic card drivers and kernels. To update your Fedora system, use the following command:

sudo dnf upgrade --refresh

    This guide uses sudo for commands that require root privileges. If your user account is not in the sudoers file yet, follow the guide on how to add a user to sudoers on Fedora before continuing.

To leverage the RPM Fusion repository for NVIDIA driver installation, add both the free and nonfree repositories. For detailed setup options and troubleshooting, see the complete RPM Fusion installation guide:

sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

The $(rpm -E %fedora) expression expands to your current Fedora release number (such as 43 or 44). Verify the repositories enabled correctly before installing the driver packages:

dnf repo list --all | grep -i rpmfusion

Example output should show the main RPM Fusion Free and Nonfree repositories enabled:

rpmfusion-free      RPM Fusion for Fedora 43 - Free      enabled
rpmfusion-nonfree   RPM Fusion for Fedora 43 - Nonfree   enabled

With RPM Fusion enabled, install the NVIDIA driver with the akmod-nvidia package so the kernel module rebuilds automatically after kernel updates:

sudo dnf install akmod-nvidia

    Do not reboot immediately after installation completes. The akmod system builds the NVIDIA kernel module in the background after DNF finishes, which takes 2-5 minutes depending on your CPU. Rebooting before the build completes will cause the system to fall back to Nouveau drivers, making it appear that the NVIDIA drivers “broke” or failed to install. Wait for the build to finish before rebooting (verification steps provided below).

If you need CUDA tools or the nvidia-smi command, install the CUDA driver tools package as well:

sudo dnf install xorg-x11-drv-nvidia-cuda

Enable NVIDIA Power Management Services

NVIDIA provides systemd services to preserve GPU state during suspend, hibernate, and resume cycles, preventing black screens or system crashes after waking from sleep. Enable these services before rebooting to ensure reliable power management:

sudo systemctl enable nvidia-suspend.service
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-resume.service

These services are particularly important for laptop users who frequently suspend their systems but benefit desktop users as well by providing more reliable wake-from-sleep behavior. The services preserve video memory allocations and GPU state across power state transitions, eliminating common “black screen after suspend” issues.

Do not reboot yet. Complete the Secure Boot steps below first if Secure Boot is enabled, then use the akmod build verification steps before the final reboot in this method.
Secure Boot and MOK Enrollment on RPM Fusion

Check whether Secure Boot is enabled before continuing. If Secure Boot is disabled, you can skip this subsection and continue with the normal reboot and verification steps.

mokutil --sb-state

Typical output is either SecureBoot enabled or SecureBoot disabled. If Secure Boot is enabled, install the required tools and generate the local akmods signing key first:

sudo dnf install kmodtool akmods mokutil openssl
sudo kmodgenca -a

    If kmodgenca -a reports an existing key pair, keep the current key unless you intentionally want to replace it. Use sudo kmodgenca -a --force only when you need to regenerate the signing key.

RPM Fusion’s akmods use the generated key at /etc/pki/akmods/certs/public_key.der to sign locally built NVIDIA modules. Enroll that key once so future driver rebuilds load under Secure Boot:

sudo mokutil --import /etc/pki/akmods/certs/public_key.der

Set a one-time password when prompted, then reboot and complete enrollment:

    The MOK enrollment screen usually uses a US QWERTY keyboard layout, even if your Fedora desktop uses a different layout. Use a simple password you can retype easily.

    On reboot, choose “Enroll MOK” on the blue enrollment screen
    Confirm the key, enter the password you set, and finish enrollment
    Reboot; the signed driver will now load under Secure Boot

After enrollment, verify driver installation with nvidia-smi (covered in the Verification section below).
Nouveau Driver Automatic Disabling

RPM Fusion’s akmod-nvidia package automatically blacklists and disables the Nouveau open-source driver. After reboot, verify Nouveau is not loaded:

lsmod | grep nouveau

This should return no output. If output appears, the NVIDIA driver installation may have failed. Check if the kmod package built successfully:

rpm -qa | grep kmod-nvidia

You should see output like kmod-nvidia-6.13.5-200.fc43.x86_64-580.119.02-1.fc43.x86_64. If no output appears, wait a few minutes for the akmod build to complete and check for errors in /var/cache/akmods/.
Wayland Support on RPM Fusion

NVIDIA drivers version 495 and newer support Wayland (Fedora 35+). RPM Fusion packages have kernel mode setting (KMS) enabled by default, which Wayland requires. No additional configuration is needed in most cases. Wayland will be available after reboot.

    GNOME upstream drops the Xorg session starting with Fedora 43, making GNOME Wayland-only. If you need Xorg for older drivers (470xx/390xx/340xx) or specific workflows, use a desktop that still offers Xorg sessions (KDE Plasma, XFCE, MATE, Cinnamon) or stick to a release that ships Xorg.

If Wayland doesn’t appear as a session option, verify KMS is enabled:

cat /proc/cmdline | grep nvidia-drm

If empty or missing nvidia-drm.modeset=1, add it:

sudo grubby --update-kernel=ALL --args='nvidia-drm.modeset=1'

Then reboot. For additional Wayland configuration details, consult the Fedora documentation on display server configuration if needed.
Desktop-Only vs CUDA Tools (Including nvidia-smi)

The base akmod-nvidia package provides the kernel driver and core graphics stack (OpenGL/Vulkan) for gaming and desktop workloads. If you need CUDA tools or plan to verify the driver with nvidia-smi, install the CUDA tools package:

sudo dnf install xorg-x11-drv-nvidia-cuda

Desktop-only users can skip this package, but note that nvidia-smi is provided by xorg-x11-drv-nvidia-cuda on RPM Fusion. You can add it later without reinstalling the base driver.
Proprietary vs Open-Source Drivers on RPM Fusion

RPM Fusion installs the standard proprietary NVIDIA kernel module path by default. NVIDIA also provides an open kernel module variant for newer GPUs, but RPM Fusion’s NVIDIA howto specifically treats it as an advanced option rather than the default recommendation.

    Proprietary driver (default akmod-nvidia): Best compatibility for all GPUs Kepler and newer. Recommended for gaming and CUDA. Full feature support.
    Open kernel module (akmod-nvidia-open): Advanced option for Turing/RTX 20-series and newer GPUs. Requires the rpmfusion-nonfree-tainted repository and is not recommended for most Fedora users by default.

If you specifically need the open kernel module variant, enable the tainted repo and switch packages:

sudo dnf install rpmfusion-nonfree-release-tainted
sudo dnf swap akmod-nvidia akmod-nvidia-open
sudo reboot

    RPM Fusion notes that most users should not switch to akmod-nvidia-open. Stay on akmod-nvidia unless you have a clear reason to test the open kernel module path on supported hardware.

Wait for Akmod Build to Complete

After the DNF transaction completes, the akmod system automatically builds the NVIDIA kernel module in the background. This process compiles the driver against your running kernel and can take 2-5 minutes depending on your CPU speed. Many users mistakenly believe NVIDIA drivers “break after every kernel update” when they simply rebooted too quickly before the build finished.
Verification Method 1: Check Module Availability

Run this command repeatedly until it returns a version number instead of an error:

modinfo -F version nvidia

Example output when build is still in progress (wait longer):

modinfo: ERROR: Module nvidia not found.

Example output when build completed successfully (safe to reboot):

580.119.02

Once you see a version number like above, the kernel module is ready and you can safely reboot.
Verification Method 2: Monitor Build Log in Real-Time

For detailed progress monitoring, watch the akmod build log as it compiles:

sudo tail -f /var/cache/akmods/nvidia/$(uname -r)-$(uname -m)/last.log

If the per-kernel log path is missing or you want a simpler view, you can monitor akmod messages in the journal instead:

sudo journalctl --follow --grep=akmod

This displays compilation output in real-time. Successful build ends with output similar to:

Building modules, stage 2.
  MODPOST 5 modules
CC [M]  /var/lib/dkms/nvidia/580.119.02/build/nvidia-drm.mod.o
  LD [M]  /var/lib/dkms/nvidia/580.119.02/build/nvidia-drm.ko
make[1]: Leaving directory '/usr/src/kernels/6.13.5-200.fc43.x86_64'
DKMS: build completed.

The key phrase is “DKMS: build completed.” Press Ctrl+C to exit the log viewer once you see this message.
Verification Method 3: Check Kmod Package Installation

After akmod completes, it automatically installs a kmod package for your kernel version. Verify it exists:

rpm -qa | grep "kmod-nvidia.*$(uname -r)"

Example output confirming successful build and installation:

kmod-nvidia-6.13.5-200.fc43.x86_64-580.119.02-1.fc43.x86_64

If this command returns nothing, the build either failed or is still in progress. Check the akmod build log (Method 2 above) for errors.
Recovery: What If You Reboot Too Early?

If you reboot before akmod finishes building the module, the system will boot without the NVIDIA driver loaded. You’ll see Nouveau driver active instead when running lspci -k | grep -A3 VGA. The fix is simple: the akmod build will complete automatically during the next boot (taking 2-3 minutes longer than usual), or you can manually trigger it with sudo akmods --force after logging in, then reboot again.
Install NVIDIA Legacy Drivers on Fedora (For Older Cards Only)

RPM Fusion still ships legacy driver branches on a best-effort basis. They can break with new kernels and are Xorg-focused, so expect more maintenance than the current driver.

Most Kepler-based GeForce 600/700 cards use the 470xx branch:

sudo dnf upgrade --refresh
sudo dnf install xorg-x11-drv-nvidia-470xx akmod-nvidia-470xx
sudo dnf install xorg-x11-drv-nvidia-470xx-cuda

    Some GTX 700-series cards (for example GTX 750, GTX 750 Ti, and some GTX 745 variants) are Maxwell-based and use the current driver branch, not 470xx. Check the GPU architecture if you are unsure.

Cards with Fermi architecture (GeForce 400/500 and some uncommon 600/700 models) use the 390xx branch:

sudo dnf upgrade --refresh
sudo dnf install xorg-x11-drv-nvidia-390xx akmod-nvidia-390xx
sudo dnf install xorg-x11-drv-nvidia-390xx-cuda

RPM Fusion’s 340xx packages are not available on Fedora 43 (verified on Fedora 43 during this review), so this guide does not recommend a 340xx install path on current Fedora releases.

For GeForce 8/9/200/300-era GPUs that require the 340xx branch, use Nouveau, an older supported Fedora release with compatible packaging, or plan a GPU upgrade. Legacy branches covered here (470xx and 390xx) are also Xorg-only. Fedora 43 and newer ship GNOME as Wayland-only without an Xorg session; choose a desktop that still ships Xorg (KDE Plasma, XFCE, MATE, Cinnamon) or use an older Fedora release that includes the GNOME Xorg session if you rely on these drivers.

    If a legacy branch fails to build or breaks after kernel updates, fall back to Nouveau (sudo grubby --update-kernel=ALL --remove-args='rd.driver.blacklist=nouveau,nova_core modprobe.blacklist=nouveau,nova_core' then sudo dracut --force and reboot) or plan a GPU upgrade.

Method 2: Install NVIDIA Drivers via NVIDIA CUDA Repository (Older Fedora Releases)

NVIDIA maintains an official CUDA repository with drivers tailored for CUDA-accelerated computing and bleeding-edge GPU support. This method provides NVIDIA’s official driver packages directly.
Fedora Version Support and Prerequisites

    NVIDIA currently publishes a CUDA repository for Fedora 42, but not Fedora 43 or newer. Use RPM Fusion on Fedora 43+ and verify current CUDA repo support at NVIDIA’s CUDA Downloads page before proceeding on older Fedora releases.

Before installing, ensure your system is updated to prevent potential conflicts between graphic card drivers and kernels:

sudo dnf upgrade --refresh

Import NVIDIA CUDA Repository

Import the repository for your Fedora version (this example uses Fedora 42; replace 42 with your version number if different):

sudo curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/fedora42/x86_64/cuda-fedora42.repo -o /etc/yum.repos.d/cuda-fedora42.repo

Install Dependencies and NVIDIA Drivers

Install the kernel headers and development tools required for DKMS module compilation. For detailed kernel header management, see the Fedora kernel headers guide:

sudo dnf install kernel-headers kernel-devel dkms

Search the repository to see available NVIDIA driver packages:

dnf search nvidia-driver kmod-nvidia

Install the NVIDIA driver with DKMS support for automatic kernel module rebuilds:

sudo dnf install nvidia-driver kmod-nvidia-latest-dkms

This installs the proprietary NVIDIA driver and configures DKMS to automatically rebuild the kernel module when your kernel updates. The installation process downloads packages, resolves dependencies, and configures the driver.

For the open-source kernel module variant (requires Turing/RTX 20-series GPU or newer):

sudo dnf install nvidia-driver kmod-nvidia-open-dkms

Most users should use the proprietary driver (kmod-nvidia-latest-dkms) for better compatibility and performance. The open-source variant is experimental and best suited for users who need kernel integration improvements on newer GPUs.
Verify CUDA Repository Driver Installation

After installation completes, reboot your system:

sudo reboot

After rebooting, verify the NVIDIA driver loaded correctly:

nvidia-smi

Successful output displays your GPU model, driver version, and CUDA version. If nvidia-smi returns “command not found”, install the CUDA utilities package:

sudo dnf install nvidia-driver-cuda

Method 3: Install NVIDIA Drivers via Runfile (Last Resort)

    The runfile method is a last resort and not recommended for most users. Fedora community members have reported several issues with this approach, including Wayland session conflicts, X configuration failures, driver persistence problems across kernel updates, and manual rebuild requirements. Only use this method if both RPM Fusion and the NVIDIA CUDA repository do not support your GPU model or Fedora version. If Secure Boot is enabled, the runfile method also requires a separate manual module-signing workflow (or temporarily disabling Secure Boot). Advanced users should be aware of compiler compatibility requirements and kernel module configuration details outlined below.

The Runfile method offers a manual approach to installing NVIDIA drivers on Fedora Linux when other methods aren’t suitable. It provides flexibility, allowing you to select any driver version directly from NVIDIA’s official website, though it requires significantly more manual work to maintain across kernel updates.
Download the NVIDIA Driver on Fedora

Begin by navigating to the NVIDIA website to download the appropriate driver version for your graphics card. The runfile method’s advantage is the ability to handpick any version, ensuring compatibility with specific needs or applications.

After downloading, the file will typically reside in the ~/Downloads directory unless you’ve chosen a different download location.
Screenshot of searching for NVIDIA drivers to download from NVIDIA's official website on Fedora LinuxHow to search for the right NVIDIA driver for your system on NVIDIA’s official website.
Screenshot showing recommended NVIDIA driver version for download on Fedora Linux Proceed to download the recommended NVIDIA driver version for your Fedora Linux system.
Disable the Nouveau Drivers (and NOVA Core if Present)

To ensure a smooth installation, disable Nouveau before running the NVIDIA installer. On newer Fedora kernels, also blacklist the optional nova_core module if present. Run:

printf "blacklist nouveau\noptions nouveau modeset=0\nblacklist nova_core\n" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf > /dev/null

This command performs two primary actions:

    blacklist nouveau: Prevents the system from auto-loading the Nouveau driver module.
    options nouveau modeset=0: Disables kernel mode-setting, a method used to set the console’s video mode. This step is crucial when installing proprietary NVIDIA drivers.
    blacklist nova_core: Blocks the newer NOVA Core module on Fedora kernels that ship it. This entry is harmless on systems where the module is not present.

Regenerate initramfs and Reboot

Before the changes take effect, regenerate the initramfs:

sudo dracut --force

Switch to CLI mode by disabling the GUI mode:

sudo systemctl set-default multi-user.target

Now reboot using the following command:

sudo reboot

Note: If you miss this step, you can temporarily switch to CLI mode using:

sudo systemctl isolate multi-user.target

Navigate to Downloaded Driver and Prepare Installation

After rebooting, you’ll be in CLI mode. Navigate to the directory containing the downloaded NVIDIA Runfile:

cd ~/Downloads

Compiler Compatibility (Fedora 42+ Users)

NVIDIA drivers 580.xx on Fedora 42+ require explicit compiler flags for kernel module compilation due to modern C standards requirements. Before running the installer, set the C compiler environment variable to ensure compatibility:

export CC="gcc -std=gnu17"

This flag ensures compatibility with modern C standards required by the NVIDIA kernel module build system. Skip this step if installing NVIDIA 470.xx or 390.xx legacy drivers.
Run the Installer

Initiate the installation process. Use full file name if you have multiple binaries in the same directory:

sudo bash NVIDIA-Linux-x86_64-*

Or with explicit version (example):

sudo bash NVIDIA-Linux-x86_64-580.119.02.run

During the installation, you’ll encounter several prompts:

32-bit Compatibility Libraries

Decide whether to install NVIDIA’s 32-bit Compatibility Libraries. While this is a user choice, installing them can benefit specific applications.
Screenshot of prompt to install NVIDIA 32-bit libraries on Fedora Linux Prompt asking for permission to install 32-bit libraries for NVIDIA drivers on Fedora.

Kernel Module Sources with DKMS

This prompt inquires if you’d like the kernel module sources to register with DKMS. Opting for ‘YES’ ensures that DKMS will auto-build a new module if your kernel updates.
Screenshot of selecting 'Yes' to register NVIDIA kernel modules with DKMS on Fedora Linux Choosing ‘Yes’ to register NVIDIA kernel modules with DKMS during the installation process.

NVIDIA X Driver Configuration

NVIDIA will ask if it should auto-run the nvidia-xconfig utility. This action updates your X configuration file, ensuring the NVIDIA X driver is utilized upon restarting. It also backs up any existing X configuration files. It’s advisable to select ‘YES’ for a seamless experience.
Screenshot of prompt to set NVIDIA xconfig utility on Fedora Linux Prompt asking to set the NVIDIA xconfig utility during driver installation on Fedora.

Upon completing these steps, you’ve successfully installed the NVIDIA drivers via the Runfile method on Fedora Linux.
Screenshot of X configuration file being updated and NVIDIA drivers installed via run file on Fedora Linux Final steps showing the X configuration file being updated and NVIDIA drivers installed.
Kernel Module Configuration (Post-Installation)

Important for display stability: The installer sets fbdev=0 in /etc/modprobe.d/nvidia.conf during runlevel 3 (CLI mode) installation to prevent framebuffer conflicts during module compilation. After successful installation and before rebooting to graphical mode, update this parameter:

## Check current fbdev setting
cat /etc/modprobe.d/nvidia.conf | grep fbdev

## If fbdev=0 is present, change to fbdev=1 for graphical mode
sudo sed -i 's/fbdev=0/fbdev=1/g' /etc/modprobe.d/nvidia.conf

The fbdev parameter controls framebuffer console access: fbdev=0 is required during kernel module compilation in CLI mode to avoid conflicts, but fbdev=1 should be set before returning to graphical mode to enable proper display console functionality in GNOME/KDE.
DKMS Compiler Persistence (Fedora 42+)

If you’re using DKMS for automatic kernel module rebuilds (recommended), you must add the compiler flag permanently to the DKMS configuration. This ensures that future kernel updates compile correctly without manual intervention:

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

This ensures that kernel module rebuilds after future kernel updates will use the correct compiler flags automatically, preventing build failures on subsequent dnf kernel updates.
Re-enable GUI and Reboot

Before concluding, switch back to the GUI login:

sudo systemctl set-default graphical.target

Finally, reboot your system:

sudo reboot

Advertisement
Verify NVIDIA Drivers on Fedora

After installing the NVIDIA drivers on your Fedora system, it’s crucial to ensure that the installation was successful and that the drivers function as expected. This verification step provides confidence in the system’s graphics performance and stability.
Verify NVIDIA Drivers with the Command Line (CLI)

The NVIDIA System Management Interface (nvidia-smi) provides a concise overview of GPU status, driver version, memory usage, and temperature.

On RPM Fusion installations, nvidia-smi is included in the xorg-x11-drv-nvidia-cuda package (not the base akmod-nvidia package). If you skipped that package earlier and see “command not found”, install it first:

sudo dnf install xorg-x11-drv-nvidia-cuda

Then run:

nvidia-smi

To check only GPU memory and compute capability:

nvidia-smi --query-gpu=compute_cap,memory.total --format=csv,noheader

To verify OpenGL hardware acceleration is working:

glxinfo | grep "OpenGL version"

This should display an OpenGL version backed by NVIDIA (not Mesa or Nouveau). Example output showing successful NVIDIA OpenGL rendering:

OpenGL version string: 4.6.0 NVIDIA 580.119.02

If it shows Mesa or software rendering instead, the NVIDIA driver may not have loaded. Verify the akmod build completed and check lsmod | grep nvidia to confirm the kernel module is loaded.

Example output showing successful NVIDIA driver installation:

+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.119.02             Driver Version: 580.119.02     CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3060      Off   | 00000000:01:00.0  On   |                  N/A |
| 30%   45C    P8              15W / 170W |     512MiB / 12288MiB  |      0%      Default |
+-----------------------------------------+------------------------+----------------------+

Accessing NVIDIA X Server Settings (GUI)

For a graphical view of your NVIDIA GPU settings, you can access the NVIDIA X Server settings application. Launch it with:

nvidia-settings

Alternatively, navigate to “Activities” → “Show Applications” and search for “NVIDIA X Server”.
Enable Hardware Video Acceleration with NVIDIA Drivers on Fedora

NVIDIA GPUs support hardware-accelerated video decoding and encoding through VDPAU and VAAPI, which significantly reduces CPU load during video playback, streaming, and encoding. Modern NVIDIA drivers (GeForce 8 series and newer) include VDPAU support automatically, but you need additional libraries to expose this capability to media players and web browsers.

Install the required acceleration libraries:

sudo dnf install libva-nvidia-driver vdpauinfo libva-utils

After installation completes, verify VDPAU support and check which video codecs your GPU can hardware-accelerate (run this after rebooting with NVIDIA drivers loaded):

vdpauinfo

The output lists supported codec profiles (H.264 for standard video, HEVC/H.265 for 4K content, VP9 for YouTube, AV1 for modern streaming, MPEG-2 for DVD/Blu-ray, and VC-1 for older media) your GPU can hardware-accelerate along with maximum resolution support for each codec. Most modern video players (VLC, MPV, Firefox, Chromium) automatically detect and use these capabilities when available, but you may need to enable hardware acceleration explicitly in browser settings (Firefox: about:preferences → Performance → “Use hardware acceleration when available”).

Hardware acceleration improves battery life on laptops by offloading video processing from the CPU to the dedicated video decode engines on your NVIDIA GPU. Desktop users benefit from reduced CPU usage during 4K video playback and multi-stream scenarios.
Advertisement
Troubleshoot Common NVIDIA Driver Issues on Fedora
NVIDIA Driver Installation and Detection Problems on Fedora

NVIDIA drivers not detected (nvidia-smi returns “command not found”): On RPM Fusion installs, first install xorg-x11-drv-nvidia-cuda because it provides nvidia-smi. Then verify the akmod build completed by checking logs with sudo journalctl -xe | grep -i nvidia and installed kernel modules with rpm -qa | grep kmod-nvidia. CUDA repo users on older Fedora releases can reinstall with sudo dnf install nvidia-driver kmod-nvidia-latest-dkms. Runfile installations require multi-user mode before reinstalling.

GPU memory or architecture not detected: If nvidia-smi shows “No devices were found”, verify your system has an NVIDIA GPU with lspci | grep -i nvidia. Check initialization errors with sudo dmesg | grep -i nvidia.

Nouveau driver still present after installation: Verify the blacklist entries exist:

grep -R -i -E 'nouveau|nova_core' /etc/modprobe.d /usr/lib/modprobe.d

Look for blacklist nouveau. Runfile installations on newer Fedora kernels may also show blacklist nova_core. If the expected blacklist entries are missing, reapply the command from the Runfile section’s “Disable the Nouveau Drivers (and NOVA Core if Present)” subsection.
NVIDIA Display and Session Issues on Fedora

X server fails to start (black screen after login):

    Runfile method: Boot into recovery mode and remove the installation with sudo /usr/bin/nvidia-uninstall, then switch to RPM Fusion.
    RPM Fusion method: Verify MOK enrollment completed if Secure Boot is enabled.
    Boot configuration issues: For advanced boot troubleshooting, use grubby to manage kernel parameters or edit /etc/default/grub directly. After changes, run sudo grub2-mkconfig -o /boot/grub2/grub.cfg to regenerate the boot configuration.

Wayland session hangs or crashes: Some driver and Wayland combinations have compatibility issues. Try these solutions:

    On Fedora 42 and earlier with GNOME, switch to X11 in the login screen’s session selector. On Fedora 43+ with GNOME (Wayland-only), use KDE Plasma, XFCE, or another desktop that offers X11 sessions.
    As an advanced test only, try the open kernel module variant (requires rpmfusion-nonfree-tainted): sudo dnf install rpmfusion-nonfree-release-tainted && sudo dnf swap akmod-nvidia akmod-nvidia-open. Switch back if it does not improve stability.

NVIDIA Kernel Module and Update Issues on Fedora

Drivers don’t persist after kernel updates: DKMS-enabled packages (akmod on RPM Fusion, latest-dkms on CUDA repository) automatically rebuild drivers. Runfile installations require manual reinstallation after each kernel update.

DKMS build fails after kernel updates: Ensure kernel headers match your running kernel:

uname -r
sudo dnf install kernel-headers-$(uname -r) kernel-devel-$(uname -r)

Then manually trigger rebuild with sudo akmods --rebuild.
Flatpak Apps Cannot Detect GPU After Driver Updates

If Flatpak applications cannot detect your NVIDIA GPU after updating drivers, even though nvidia-smi works correctly, the Flatpak NVIDIA runtime libraries need updating separately. This issue occurs because Flatpak maintains its own copy of NVIDIA userland libraries that must match your system driver version.

First, verify your system driver version is working:

nvidia-smi

If the GPU appears correctly, check your Flatpak NVIDIA runtime version:

flatpak list | grep org.freedesktop.Platform.GL.nvidia

Example output showing version mismatch:

nvidia-580-119-02	org.freedesktop.Platform.GL.nvidia-580-119-02		1.4	system

If the version number doesn’t match your installed driver (for example, Flatpak shows 580-105-08 but nvidia-smi shows 580.119.02), the mismatch prevents Flatpak apps from detecting your GPU. Update Flatpak packages:

flatpak update

After updating, reboot your system. The Flatpak NVIDIA libraries will now match your system driver version, and GPU-accelerated applications should detect your hardware correctly.

    After upgrading NVIDIA drivers through DNF, always run flatpak update separately before rebooting. System package managers (DNF, RPM) do not automatically update Flatpak runtimes. Skipping this step leaves Flatpak apps unable to communicate with your GPU until the runtime versions sync.

External Monitor and HDMI Detection Issues (Optimus Laptops)

Laptops with NVIDIA Optimus (dual GPU configuration) may experience external monitor detection failures where HDMI or DisplayPort outputs don’t recognize connected displays at boot but work correctly when plugged in after the system finishes booting. This behavior typically indicates BIOS security setting conflicts with NVIDIA driver initialization.
Verify Driver Loading and Module Status

First, verify the NVIDIA driver loaded successfully and can communicate with the GPU:

nvidia-smi

If this command returns “NVIDIA-SMI has failed because it couldn’t communicate with the NVIDIA driver”, the driver loaded but cannot initialize the GPU hardware properly. Check if the driver modules loaded:

lspci -nnk | grep -iA3 vga

Successful output shows the NVIDIA GPU with kernel driver loaded:

01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA106M [GeForce RTX 3060 Mobile / Max-Q] [10de:2560] (rev a1)
	Subsystem: Dell Device [1028:0b5f]
	Kernel driver in use: nvidia
	Kernel modules: nouveau, nvidia_drm, nvidia

If “Kernel driver in use” shows “nouveau” or is missing entirely, the NVIDIA driver isn’t loaded. Force rebuild the kernel modules:

sudo akmods --rebuild --force
sudo reboot

BIOS Settings That Affect Display Detection

If external monitors still don’t detect at boot, check your BIOS/UEFI settings for these common conflicts:

    Secure Boot: While RPM Fusion drivers sign modules automatically, some BIOS implementations block signed third-party modules anyway. Disable Secure Boot temporarily to test.
    Fast Boot / Quick Boot: This feature skips hardware initialization steps that NVIDIA drivers need for external display detection. Disable Fast Boot in BIOS power management settings.
    TPM (Trusted Platform Module): Some implementations interfere with GPU initialization timing. Disable TPM to test if it resolves detection issues.
    CSM (Compatibility Support Module) / Legacy Boot: Mixed UEFI/Legacy settings can cause display output routing problems. Ensure pure UEFI mode without CSM enabled.

After changing BIOS settings, reboot fully (cold boot, not warm restart) to ensure hardware initialization runs completely.
Common Symptoms of Initialization Problems

Check for these secondary symptoms that indicate BIOS conflicts:

    System freezes when screen powers off with HDMI connected: The driver cannot handle display power management events properly, typically caused by incomplete hardware initialization at boot.
    Shutdown hangs with external monitor connected: The driver cannot release external display resources cleanly, another symptom of initialization timing issues.
    Boot message showing fallback to Nouveau: Even with akmod-nvidia installed, BIOS security features may prevent module loading during early boot. Check for this message:

nvidia kernel modules not found, falling back to nouveau

This indicates module signing and Secure Boot conflicts that need resolution.

    Dell, Lenovo, and HP gaming laptops frequently ship with aggressive security defaults that conflict with third-party kernel modules. If disabling Secure Boot alone doesn’t resolve HDMI detection, systematically disable other security features (Intel SGX, Virtualization-based Security, TPM) one at a time until you identify the specific conflict.

Resolution Detection Failures and Driver Version Conflicts

After system updates, monitors may fail to detect their native resolutions (for example, 2K or 4K displays limited to 1920×1080) even though the NVIDIA driver appears to load correctly. This typically indicates either corrupted EDID data, conflicting kernel module versions, or known bugs in specific driver releases.
Check Kernel Module Conflicts

Check for multiple kmod versions installed simultaneously:

rpm -qa | grep kmod-nvidia

Example output showing multiple conflicting versions:

kmod-nvidia-6.12.8-200.fc43.x86_64-580.119.02-1.fc43.x86_64
kmod-nvidia-6.13.4-200.fc43.x86_64-580.119.02-1.fc43.x86_64
kmod-nvidia-6.13.5-200.fc43.x86_64-580.119.02-1.fc43.x86_64

If you see multiple versions like above, the system may load the wrong module version. Verify which kernel is running:

uname -r

Then confirm the matching kmod package exists for your running kernel. If the kmod version doesn’t match your kernel version, remove conflicting old modules and rebuild:

sudo dnf remove $(rpm -qa | grep 'kmod-nvidia.*6.12.8')
sudo akmods --force
sudo dracut --force
sudo reboot

Replace 6.12.8 with the old kernel version you want to remove. Only remove kmod packages for kernels you no longer boot into.
Known Driver Version Issues

Driver version issues with high-resolution monitors: Some driver versions have reported bugs affecting 2K (2560×1440), ultra-wide (3440×1440), and 4K displays where maximum resolution detection fails or displays limited resolution options. If you recently upgraded and lost resolution options, downgrade to the previous stable release:

sudo dnf downgrade akmod-nvidia xorg-x11-drv-nvidia
sudo akmods --force
sudo reboot

After downgrading, verify the driver version:

modinfo -F version nvidia

Expected output confirming successful downgrade:

580.119.02

If the version matches the previous stable version, reboot and test your monitor’s native resolution in display settings.

nvidia-smi missing despite drivers installed: If nvidia-smi returns “command not found” even though rpm -qa | grep nvidia shows installed packages, the RPM Fusion CUDA tools package is missing. Install it explicitly:

sudo dnf install xorg-x11-drv-nvidia-cuda

This package provides nvidia-smi and other GPU management utilities. After installation, verify:

which nvidia-smi
nvidia-smi

Expected output confirming nvidia-smi is now available:

/usr/bin/nvidia-smi

The which command should return the path above, confirming the utility is installed. Running nvidia-smi will then display your GPU information.
EDID and Session Type Troubleshooting

If resolution detection still fails after verifying drivers and removing conflicts, check for EDID detection errors:

sudo journalctl -b | grep -i edid
xrandr --verbose

EDID errors indicate the monitor isn’t communicating its capabilities correctly. Try a different cable (DisplayPort vs HDMI) or test with another monitor to isolate hardware issues from driver problems. Additionally, Wayland sessions may not detect all resolutions that X11 sessions expose; test both session types from the login screen to compare available resolutions.
GUI Package Manager Reboot Timing Issues

Users who update Fedora through graphical package managers (GNOME Software, KDE Discover) may experience a specific timing issue that creates the false impression that NVIDIA drivers “break with every kernel update.” This section explains why this happens and how to avoid it.
Understanding the Problem

GUI package managers like KDE Discover prompt you to reboot immediately after the DNF transaction completes installing packages. However, the akmod background process that compiles the NVIDIA kernel module starts after DNF finishes and takes an additional 2-5 minutes. If you click “Reboot Now” when prompted, the system reboots before akmod completes, causing the NVIDIA driver to fail loading on the next boot.

Symptoms of premature reboot:

    Black screen or low-resolution display after kernel updates
    lsmod | grep nvidia returns nothing (driver not loaded)
    lspci -k | grep -A3 VGA shows “Kernel driver in use: nouveau” instead of nvidia
    System appears to “forget” NVIDIA drivers were installed

Solution Options

Solution 1: Use command-line updates (recommended for reliability)

Instead of relying on GUI package managers for system updates, use DNF directly in a terminal:

sudo dnf upgrade --refresh

After DNF completes, wait for akmod to finish building (verify with modinfo -F version nvidia as described in the “Wait for Akmod Build to Complete” section above), then reboot manually:

sudo reboot

Solution 2: Ignore premature reboot prompts from GUI updaters

If you prefer using GNOME Software or KDE Discover for updates, when you see the “Restart Now” or “Reboot Required” notification after updates complete, do NOT click it immediately. Instead:

    Dismiss the reboot notification
    Open a terminal and run modinfo -F version nvidia
    Wait until it returns a version number instead of “Module nvidia not found”
    Only then reboot your system

Solution 3: Monitor CPU activity after updates

After GUI package managers finish updating packages, open System Monitor (GNOME) or KSysGuard (KDE) and watch CPU usage. You’ll see sustained CPU activity (one or more cores at 100%) for 2-5 minutes while akmod compiles the NVIDIA module. Wait until CPU usage returns to idle levels before rebooting.

What if you already rebooted too early?

If you’ve already rebooted and the NVIDIA driver isn’t loaded, don’t reinstall anything. The akmod build will complete automatically during boot (taking 2-3 minutes longer than usual), or you can manually trigger it:

sudo akmods --force
sudo reboot

This forces immediate module compilation without waiting for the next boot cycle.
Wayland Performance and Compatibility Issues with NVIDIA on Fedora

NVIDIA’s Wayland support has improved significantly with driver 555.42+ and explicit sync protocol support. However, if you experience flickering, lag, or performance issues on Wayland, these fixes apply to all installation methods (RPM Fusion, CUDA repository, and runfile).
Disable GSP Firmware (KDE Plasma Performance)

The GPU System Processor (GSP) firmware can cause performance degradation in KDE Plasma 6.0+ despite being newer (driver 510+). Consequently, if you experience frame drops or stuttering in KDE Plasma, first check if GSP firmware is enabled:

nvidia-smi -q | grep "GSP Firmware"

Example output when GSP firmware is enabled (causing performance issues):

    GSP Firmware Version                  : 580.119.02

If output shows a version number like above instead of “N/A”, GSP is enabled and may be causing performance degradation in KDE Plasma. Disable it by adding a kernel parameter with grubby (works on both UEFI and BIOS installs), then reboot:

sudo grubby --update-kernel=ALL --args='nvidia.NVreg_EnableGpuFirmware=0'
sudo reboot

Enable Preserve Video Memory Allocations (Suspend/Resume)

If you experience graphical glitches, artifacts, or black screens after suspend/resume, enable the preserve video memory allocations module parameter. First, verify the current status:

sudo grep "PreserveVideoMemoryAllocations" /proc/driver/nvidia/params

Example output showing disabled state (causes suspend/resume issues):

PreserveVideoMemoryAllocations: 0

If output shows “0” like above, the feature is disabled and may cause graphical artifacts after suspend/resume. Enable it by adding the kernel parameter with grubby and reboot:

sudo grubby --update-kernel=ALL --args='nvidia.NVreg_PreserveVideoMemoryAllocations=1'
sudo reboot

Verify the setting applied:

sudo grep "PreserveVideoMemoryAllocations" /proc/driver/nvidia/params

Expected output confirming the setting is now enabled:

PreserveVideoMemoryAllocations: 1

The “1” confirms video memory preservation is active. Suspend and resume your system to verify the graphical glitches no longer occur.
VRR/FreeSync/G-SYNC Screen Flickering and Black Screens

High-refresh-rate monitors with Variable Refresh Rate (VRR), FreeSync, or G-SYNC enabled frequently experience severe flickering and black screen flashes on Wayland with NVIDIA drivers. This issue affects both NVIDIA and AMD GPUs and is particularly noticeable when alt-tabbing out of games or during static content like loading screens.

Symptoms of VRR-related flickering:

    Monitor briefly goes black (1-2 seconds) when switching windows or applications
    Heavy flickering during gaming, especially in menus or loading screens
    Screen turns black repeatedly when alt-tabbing out of full-screen games
    Flickering on static content but smooth during motion (counterintuitive VRR behavior)
    Issue disappears completely when using X11 session instead of Wayland

Work through these fixes in order, starting with the display session change because it is the fastest and most reliable workaround.
Switch to X11 Session (Immediate Fix)

The most reliable workaround is using X11 instead of Wayland. At the login screen (GDM or SDDM), click the gear icon before entering your password and select “GNOME on Xorg” (Fedora 42 and earlier) or “Plasma (X11)” (available on all versions). Fedora 43+ ships GNOME without an Xorg session, so GNOME users on Fedora 43 must use KDE Plasma, XFCE, or another desktop for X11 access. X11 handles VRR and G-SYNC properly with NVIDIA drivers without flickering issues.
Disable VRR in Desktop Settings (Wayland)

If you must use Wayland, disable Variable Refresh Rate in your desktop environment settings:

For GNOME 47+:

    Open Settings → Displays
    Click on your monitor
    Toggle “Variable Refresh Rate” to OFF
    Apply changes and reboot

For KDE Plasma 6+:

    Open System Settings → Display and Monitor
    Select your monitor
    Uncheck “Enable Adaptive Sync”
    Apply and reboot

Disabling VRR eliminates the flickering but you lose adaptive refresh rate benefits. Fixed refresh rate (144Hz, 165Hz, etc.) will still work normally.
Create Custom EDID Table (Advanced Users)

Some monitors report incorrect VRR frequency ranges in their EDID data, causing the driver to switch refresh rates inappropriately. Extract your monitor’s EDID, modify the VRR frequency range (for example, increase minimum from 24Hz to 48Hz), and reload the custom EDID. This advanced procedure requires familiarity with edid-decode, hex editors, and kernel parameters.

First, install the monitor-edid package and check your current VRR range:

sudo dnf install monitor-edid
sudo monitor-get-edid | edid-decode | grep -A5 "Variable Refresh"

If your monitor reports a very low minimum VRR frequency (24Hz-30Hz), this may trigger excessive mode switching. Increasing the minimum to 48Hz often resolves flickering while maintaining VRR functionality. Detailed EDID editing tutorials are available in NVIDIA and kernel documentation, but this method requires significant technical expertise.
Test Open-Source Kernel Modules

If you have a Turing (RTX 20-series) or newer GPU and need an advanced troubleshooting option, test the open NVIDIA kernel modules to see whether VRR timing improves:

sudo dnf install rpmfusion-nonfree-release-tainted
sudo dnf swap akmod-nvidia akmod-nvidia-open
sudo reboot

After rebooting, test VRR behavior. If flickering persists, switch back to the default package path with sudo dnf swap akmod-nvidia-open akmod-nvidia.

    VRR/G-SYNC flickering on Wayland remains a known limitation affecting both NVIDIA and AMD GPUs as of February 2026. The Wayland VRR stack is still maturing. For gaming and high-refresh-rate workflows, X11 sessions usually provide more stable VRR behavior until compositor support improves.

Chrome and Chromium Wayland Support

For Google Chrome, Chromium, and Edge browsers to run smoothly on Wayland with NVIDIA, enable the Ozone platform flag. Open the browser and navigate to chrome://flags (or edge://flags), search for “Preferred Ozone platform”, and set it to either “Wayland” or “auto”. Restart the browser.

For Visual Studio Code and other Electron apps, launch with explicit Wayland flags. Example for VS Code:

code --enable-features=UseOzonePlatform,WaylandWindowDecorations --ozone-platform-hint=auto

To make this permanent, create or edit ~/.config/code-flags.conf and add the flags.
NVIDIA Video Codec and Media Player Issues on Fedora

Black screens in VLC, Firefox, or other media players with NVIDIA drivers are typically caused by missing codecs (openh264, ffmpeg), not driver issues. This affects all installation methods. To resolve this, install the necessary codec packages:

sudo dnf config-manager setopt fedora-cisco-openh264.enabled=1
sudo dnf install openh264 mozilla-openh264 libavcodec-freeworld ffmpeg gstreamer1-plugins-bad-freeworld gstreamer1-plugins-ugly

For RPM Fusion users, these packages require the Free and Nonfree repositories to be enabled (covered in the RPM Fusion installation section). After installation, restart your media player or browser so the codec libraries load correctly.
LUKS-Encrypted Fedora Setup for NVIDIA Drivers (Before Final Reboot)

If your Fedora installation uses LUKS disk encryption, perform these steps before rebooting after driver installation (applies to runfile method primarily, but relevant for CUDA repository if you encounter boot issues). This ensures NVIDIA drivers load correctly during early boot.

Create or edit the dracut NVIDIA configuration:

sudo nano /etc/dracut.conf.d/nvidia.conf

Add the following line to include NVIDIA modules in the initramfs:

add_drivers+=" nvidia nvidia_modeset nvidia_uvm nvidia_drm "

Save the file (Ctrl+X, then Y, then Enter), then rebuild the initramfs:

sudo dracut --force

After this completes, you can safely reboot. The NVIDIA modules will be available during early boot, allowing LUKS password entry and driver loading in the correct sequence.
AMD and NVIDIA Hybrid Laptop Boot Issues on Fedora

Laptops with both AMD integrated graphics and NVIDIA discrete GPUs may experience boot hangs where the system freezes before reaching the login screen, even though X11 sessions launched manually with startx work correctly. This typically indicates Wayland compositor initialization failures rather than driver problems.

If you encounter this issue, first verify the NVIDIA driver loaded successfully by booting to a text console (press Ctrl+Alt+F3 at boot) and checking:

lsmod | grep nvidia
nvidia-smi

The lsmod output should show the nvidia modules loaded (nvidia, nvidia_modeset, nvidia_uvm, nvidia_drm). If empty, the driver failed to load. The nvidia-smi command should display your GPU model and driver version.

If the driver loads but graphical sessions fail, try these recovery steps in order:

1. Downgrade to previous stable driver version: Some driver releases have known issues with hybrid GPU systems. If problems started after a recent driver update, downgrade to the previous stable version:

sudo dnf downgrade akmod-nvidia xorg-x11-drv-nvidia

Wait for the akmod rebuild to complete (monitor with sudo tail -f /var/cache/akmods/nvidia/$(uname -r)-$(uname -m)/last.log), then reboot.

2. Rebuild initramfs without host-only mode: If boot hangs persist, rebuild the initial ramdisk to include all necessary modules:

sudo dracut --force --no-hostonly --kver $(uname -r)

This creates a larger initramfs (around 150MB) that includes NVIDIA modules for early boot. Reboot after completion.

3. Check for D-Bus and systemd service failures: Boot issues that show GDM restart loops or “Failed to list cached users: GDBus.Error” messages indicate deeper system service problems beyond drivers. Check failed units:

sudo systemctl --failed
sudo journalctl -b -p err

If multiple system services fail (auditd, firewalld, akmods), the issue may stem from filesystem corruption or SELinux context problems rather than NVIDIA drivers. Consider performing filesystem relabeling from rescue mode.

4. Try the open-source kernel modules: Verify which NVIDIA module variant is active:

modinfo -l nvidia

If output shows “NVIDIA” instead of “Dual MIT/GPL”, you’re using proprietary modules. RPM Fusion automatically selects open modules for compatible GPUs (RTX 20-series and newer). The open modules sometimes resolve hybrid GPU initialization timing issues.

    If your system won’t boot at all, use a Fedora Live USB to mount your root partition and chroot into it. From there, you can downgrade drivers, rebuild initramfs, check logs with journalctl --list-boots, and disable problematic services before rebooting into your installed system.

Manage Dual GPUs on Fedora Laptops (Integrated and NVIDIA)

Laptops with both integrated graphics (Intel or AMD) and NVIDIA discrete GPUs benefit from proper GPU switching management. Fedora provides switcherooctl through the switcheroo-control package to query and control GPU selection. Install it if not already present:

sudo dnf install switcheroo-control
switcherooctl

This command displays available GPUs and current power states. To launch applications on the discrete NVIDIA GPU for better performance:

switcherooctl launch -g 1 application-name

Alternatively, use NVIDIA’s PRIME offloading environment variables (works regardless of GPU management method):

__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia application-name

For persistent GPU selection preferences, modern desktop environments (GNOME 47+, KDE Plasma 6+) provide graphical GPU preference controls in application settings. Additionally, verify your power management settings to ensure the discrete GPU powers down when idle to conserve battery life.
Advertisement
Remove NVIDIA Drivers on Fedora

Use these commands if you want to remove NVIDIA drivers completely or reset the system before switching installation methods. The earlier cleanup section is still the best pre-reinstall checklist; this section is the quick removal reference.
Remove RPM Fusion or CUDA Repository Driver Packages

    Review the installed NVIDIA package list before removal so you do not remove unrelated packages by accident.

dnf list installed 'nvidia*'
sudo dnf remove akmod-nvidia akmod-nvidia-open xorg-x11-drv-nvidia\* nvidia-settings --allowerasing

Verify the driver packages are removed before switching methods:

rpm -qa | grep -E 'akmod-nvidia|xorg-x11-drv-nvidia'

If the command returns no output, the NVIDIA driver packages from RPM Fusion are removed.

If you also added NVIDIA’s CUDA repository, remove the repo file and imported keys:

sudo rm /etc/yum.repos.d/cuda-fedora*.repo
sudo rm /etc/pki/rpm-gpg/RPM-GPG-KEY-NVIDIA*

Remove Runfile Installations

For NVIDIA runfile installs, use the bundled uninstaller instead of dnf:

sudo /usr/bin/nvidia-uninstall

If you also installed the CUDA toolkit with a runfile, remove it separately (replace X.Y with your version):

sudo /usr/local/cuda-X.Y/bin/cuda-uninstall

Frequently Asked Questions
Are proprietary NVIDIA drivers available in Fedora’s default repositories?

No. Fedora’s default repositories do not ship the proprietary NVIDIA driver. Most users should install NVIDIA drivers from RPM Fusion. NVIDIA’s CUDA repository is a separate option only for older Fedora releases when NVIDIA publishes a matching Fedora repo.
Why is nvidia-smi missing after installing akmod-nvidia on Fedora?

On RPM Fusion installs, the nvidia-smi command is provided by xorg-x11-drv-nvidia-cuda, not akmod-nvidia. Install xorg-x11-drv-nvidia-cuda, then run nvidia-smi again after the kernel module build finishes.
What causes the “NVIDIA kernel module missing. Falling back to nouveau” message on Fedora?

This usually means the akmod build did not finish before reboot, or the NVIDIA kernel module failed to build for the current kernel. Wait for the build to complete, verify with modinfo -F version nvidia and rpm -qa | grep kmod-nvidia, then reboot again.
Can I use Wayland with NVIDIA drivers on Fedora 43?

Yes. Modern NVIDIA drivers support Wayland on Fedora 43, and GNOME on Fedora 43 is Wayland-only. If you need an X11 session for specific workflows or legacy drivers, use a desktop environment that still offers X11, such as KDE Plasma or XFCE.
