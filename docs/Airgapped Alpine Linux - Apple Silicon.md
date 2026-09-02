# 🛡️ Hardened Alpine Linux Build Guide: Apple Silicon (M1/M2/M3/M4)

**Hardened RAM-Only OS for Apple M-Series Macs**

This guide provides a workflow for creating a hardened Alpine Linux environment on Apple Silicon. It covers both an internal NVMe installation and a bootable USB implementation.

> [!WARNING]
> **CRITICAL RISK:** This process involves modifying the internal NVMe partition table of your Mac. While the Asahi installer is designed to be safe, a power failure or mistake during partitioning can result in a device that requires a "DFU Restore" using another Mac and Apple Configurator. **Backup your data before proceeding.**

## 🎯 Project Goal

To create a "zero-trust" runtime environment on Apple Silicon where the OS resides in RAM, utilizing the Asahi/U-Boot boot chain to maintain the highest possible security posture.

### The Boot Architecture:
1. **m1n1:** A tiny bootloader that handles the transition from Apple's proprietary environment to a standard one.
2. **U-Boot:** The "Universal Bootloader" that allows us to boot a standard Linux kernel.
3. **Alpine (Diskless):** The OS is loaded into a RAM disk (`tmpfs`), meaning no data is written to the NVMe during runtime.

---

## 🛠️ Phase 1: System Bootstrap (The Build Machine)

This phase prepares your Mac to build the Alpine image using containers.

```zsh
# Install Homebrew if missing
if ! command -v brew &> /dev/null; then
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
fi

brew update
echo "Installing OrbStack and GNU tools..."
brew install --cask orbstack
brew install coreutils gnu-sed

export PATH="/opt/homebrew/bin:$PATH"
echo "✅ Build Environment Ready."
```

### 2. Workspace Setup

```zsh
mkdir -p ~/alpine-m-series/{boot,cache,ovl_root,app_assets}
cd ~/alpine-m-series
```

---

## 🛠️ Phase 2: Creating the Hardened Rootfs

We will use a Docker container to "stage" the Alpine filesystem. Since we want this to be airgapped, we follow the same `apkovl` (overlay) logic as the Pi guide.

### 1. Fetching Packages
This downloads the minimal set of tools needed for a functional, hardened GUI.

```zsh
docker run --rm -v $(pwd):/work -w /work --platform linux/arm64 alpine:latest sh -c "
  apk update && \
  apk fetch --recursive -o /work/cache \
    alpine-base \
    cage \
    chromium \
    font-dejavu \
    mesa-dri-gallium \
    wayland-protocols \
    eudev \
    httpd
"
```

### 2. The Hardened Overlay (`apkovl`)
We now create the configuration that will be injected into RAM at boot.

**2.1 Privilege Separation & App Setup**
```zsh
mkdir -p ovl_root/home/entropylab
mkdir -p ovl_root/var/www/entropylab
cp -R app_assets/* ovl_root/var/www/entropylab/
chmod -R 755 ovl_root/var/www/entropylab
```

**2.2 Startup Service (The Kiosk)**
This script ensures that when Alpine boots, it immediately launches the browser as a low-privileged user.

```zsh
mkdir -p ovl_root/etc/init.d
cat << 'EOF' > ovl_root/etc/init.d/entropylab
#!/sbin/openrc-run
name="EntropyLab App"

depend() {
    after localmount
    keyword -jail
}

start() {
    ebegin "Starting Hardened Kiosk"
    # Create unprivileged user
    if ! id -u entropylab > /dev/null 2>&1; then
        adduser -D -u 1000 -s /bin/ash entropylab
    fi
    
    export XDG_RUNTIME_DIR=/tmp/runtime-root
    mkdir -p $XDG_RUNTIME_DIR
    chown -R entropylab:entropylab $XDG_RUNTIME_DIR
    chmod 0700 $XDG_RUNTIME_DIR
    
    # Local web server for SOP sandbox
    httpd -p 127.0.0.1:8080 -h /var/www/entropylab -u nobody:nobody
    
    # Launch Chromium via Cage (Wayland)
    su - entropylab -c "
      export XDG_RUNTIME_DIR=/tmp/runtime-root
      cage -d -- chromium-browser \
        --incognito --no-first-run --disable-sync \
        --user-data-dir=/tmp/chrome \
        http://127.0.0.1:8080/entropylab.html &
    "
    eend $?
}
EOF
chmod +x ovl_root/etc/init.d/entropylab
```

**2.3 Finalizing the Overlay**
```zsh
mkdir -p ovl_root/etc/runlevels/{sysinit,default}
ln -s /etc/init.d/entropylab ovl_root/etc/runlevels/default/entropylab
tar -czf boot/localhost.apkovl.tar.gz -C ovl_root .
```

---

## 🛠️ Phase 3: The Bootloader Installation

**This is the most critical part.** You cannot simply "copy" a bootloader to a Mac. You must use the Asahi installer to create the required partitions.

### Option A: Internal NVMe Installation
1. **Run the Asahi Installer:**
   Open your terminal in macOS and run:
   ```zsh
   curl https://alx.sh | sh
   ```
2. **Follow the prompts:** 
   - Choose the size for the Linux installation (since we are using a RAM-disk, you only need ~10GB).
   - The installer will resize your macOS partition and create the **m1n1** and **U-Boot** partitions.
   - **STOP** when the installer asks to install the full OS (Fedora/Asahi). You only need the bootloader chain.
3. **Boot into the newly created Linux partition.**
4. **Replace the Rootfs:**
   Once booted into the temporary Asahi environment, mount the root partition and replace the contents with your `cache` and `boot` folders from the `~/alpine-m-series` directory.

### Option B: Bootable USB Installation
To make a USB bootable on M-series, you still need the Mac's internal NVRAM to point to a bootloader.

1. **Prepare the USB:**
   Format a USB drive as **FAT32 (MBR)**.
2. **Copy the Chain:**
   You must copy the `m1n1` binary and the `U-Boot` binary to the root of the USB. 
   *(Note: These binaries are specific to your M-chip version—M1 vs M2. You can extract these from the Asahi installer's build artifacts).*
3. **The Alpine Image:**
   Copy the `cache` and `boot` folders (containing your `localhost.apkovl.tar.gz`) to the USB.
4. **Booting:**
   - Shut down the Mac.
   - Hold the **Power Button** until "Loading Startup Options" appears.
   - Select the USB drive.

---

## 🛠️ Phase 4: Final Hardening (The Mac Way)

To complete the "Airgapped" nature of the build, we must disable the Mac's wireless radios.

### 1. Disabling Network in Kernel
Since we are using a custom Alpine build, we modify the `cmdline.txt` (or the U-Boot script) to disable the network stack:

```zsh
# Edit your boot arguments to include:
# ip=off
```

### 2. U-Boot Lockdown
If you have access to the U-Boot prompt (by hitting a key during boot), you can disable the network boot attempts to speed up boot time and increase security:
```bash
setenv bootdelay 0
saveenv
```

---

## 📋 Summary Checklist for the User

| Step | Action | Purpose |
| :--- | :--- | :--- |
| **1** | Run OrbStack Script | Prepares the "factory" to build the OS. |
| **2** | Build `apkovl` | Creates the secure, RAM-only configuration. |
| **3** | Run `alx.sh` | Safely carves out space on the Mac NVMe for Linux. |
| **4** | Deploy Alpine | Moves the RAM-disk assets into the Asahi partition. |
| **5** | Boot & Lock | Sets `ip=off` to ensure the device is airgapped. |

### 🚀 Troubleshooting
- **Stuck at Apple Logo?** Your `m1n1` version likely doesn't match your chip (e.g., using M1 bootloader on M2). Re-run the Asahi installer.
- **OOM (Out of Memory)?** Apple Silicon Macs usually have 8GB+ RAM, but if using a very lean model, ensure you aren't loading unnecessary packages into the `apk fetch` step.
- **No GPU Acceleration?** Ensure `mesa-dri-gallium` is included in the package fetch; otherwise, Chromium will be extremely slow.
