# 🛡️ Airgapped Alpine Linux Build Guide

**Hardened RAM-Only OS for Raspberry Pi 4/5**

This guide creates a bootable microSD / USB (or a distributable `.img`) that boots Alpine Linux into RAM and launches only the EntropyLab web app in a hardened Chromium window.

Optimized for **Apple Silicon Macs** (M1/M2/M3/M4).

> [!CAUTION]
> **Hardware Requirement:** Raspberry Pi 4 or 5 with **at least 2 GB of RAM**. The entire OS and packages load into a RAM disk. 1 GB models will usually run out of memory and fail to boot.

---

## 🎯 What this build does

1. Boots Alpine fully into RAM (diskless).
2. Disables Wi-Fi and Bluetooth at firmware level.
3. Removes network / wireless / Bluetooth drivers **and** their firmware from the kernel modloop.
4. Turns off the kernel IP stack (`ip=off`).
5. Runs Chromium as an unprivileged user with strong browser hardening flags.
6. Serves `entropylab.html` from a local read-only web server (no `file://` privileges).

---

## 🛠️ Build Instructions (macOS)

### Before you start

1. Have your finished `entropylab.html` file ready.
2. Have a microSD card or USB stick you are willing to erase (Option A), **or** just generate an `.img` file (Option B).

---

### 1. Create a working folder and open Terminal there

You can put the build folder anywhere you like (Desktop, Documents, external drive, etc.).

**Recommended way (macOS Finder):**

1. Create a new folder, for example `EntropyLab-Build`.
2. Right-click the folder → **New Terminal at Folder**  
   (or open Terminal and drag the folder onto the Terminal icon).

All commands below assume you are already inside that folder.

**Tip:** Avoid spaces in the folder path if possible (e.g. prefer `EntropyLab-Build` over `Entropy Lab Build`). The commands quote paths correctly, but paths without spaces are simpler.

---

### 2. System Bootstrap (one-time tools)

This installs Homebrew (if needed), OrbStack (lightweight Docker), and GNU tools.

> **Password note:** The Homebrew install will ask for your **macOS User Administrator Password**.

```zsh
# Must be Apple Silicon
if [ "$(uname -m)" != "arm64" ]; then
  echo "❌ This workflow requires an Apple Silicon Mac."
  exit 1
fi

# Soft warning if the current path contains spaces
if [[ "$PWD" == *" "* ]]; then
  echo "⚠️  Warning: Your build folder path contains spaces."
  echo "   This usually works, but a path without spaces is simpler."
fi

# Install Homebrew if missing
if ! command -v brew >/dev/null 2>&1; then
  /bin/bash -c "$(curl -fsSL [https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh](https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh))"
fi

export PATH="/opt/homebrew/bin:$PATH"

brew update
echo "Installing OrbStack and GNU tools..."
brew install --cask orbstack
brew install coreutils gnu-sed

echo "✅ System Bootstrap complete."

```

---

### 3. Create folders and place your HTML file

```zsh
mkdir -p boot cache ovl_root app_assets

```

**Important – put your app here now:**

1. Rename your main HTML file to exactly `entropylab.html`.
2. Copy or move it into the `app_assets` folder that was just created.

The finished path must be:

```text
./app_assets/entropylab.html

```

**Safety check** (stops with a clear message if the file is missing):

```zsh
if [ ! -f app_assets/entropylab.html ]; then
  echo "❌ Missing required file: app_assets/entropylab.html"
  echo "   Rename your HTML file to entropylab.html and place it in the app_assets folder, then re-run this check."
  exit 1
fi
echo "✅ entropylab.html found."

```

---

### 4. Start OrbStack and download packages

```zsh
if ! command -v docker >/dev/null 2>&1; then
  echo "❌ Docker command not found. OrbStack may not be installed correctly."
  echo "   Try opening OrbStack from Applications, then re-run this step."
  exit 1
fi

echo "🐳 Launching OrbStack..."
open -a OrbStack 2>/dev/null || {
  echo "⚠️  Could not auto-launch OrbStack. Checking if Docker is already running..."
}

RETRY=0
MAX_RETRY=30
until docker info >/dev/null 2>&1; do
  if [ $RETRY -ge$MAX_RETRY ]; then
    echo "❌ Docker engine did not become ready after 30 seconds."
    echo "   Open OrbStack manually from Applications and run this step again."
    exit 1
  fi
  echo "⏳ Waiting for Docker... ($((RETRY+1))/$MAX_RETRY)"
  sleep 1
  RETRY=$((RETRY + 1))
done
echo "✅ Docker engine is ready."

docker run --rm -v "$(pwd):/work" -w /work --platform linux/arm64 alpine:latest sh -c "
  apk update && \
  apk fetch --recursive -o /work/cache \
    alpine-base \
    busybox-extras \
    cage \
    chromium \
    font-dejavu \
    mesa-dri-gallium \
    wayland-protocols \
    eudev \
    eudev-openrc
"

```

---

### 5. Build the hardened overlay (apkovl)

#### 5.1 Basic structure

```zsh
mkdir -p ovl_root/home/entropylab

```

#### 5.2 Auto-mount script for extra USB / SD cards

```zsh
mkdir -p ovl_root/etc/udev/rules.d
cat << 'EOF' > ovl_root/etc/udev/rules.d/99-automount.rules
ACTION=="add", SUBSYSTEM=="block", ENV{DEVTYPE}=="partition", RUN+="/usr/local/bin/auto-mount.sh %N"
EOF

mkdir -p ovl_root/usr/local/bin
cat << 'EOF' > ovl_root/usr/local/bin/auto-mount.sh
#!/bin/sh
DEVNAME=$1

BUS=$(udevadm info --query=property --name="$DEVNAME" | grep "ID_BUS=" | cut -d'=' -f2)

# Skip if already mounted (boot device)
if mount | grep -q "$DEVNAME"; then
  exit 0
fi

if [ "$BUS" = "mmc" ]; then
  mkdir -p /mnt/sdcard
  mount -o uid=1000,gid=1000,umask=000 "$DEVNAME" /mnt/sdcard
elif [ "$BUS" = "usb" ]; then
  LABEL=$(blkid -s LABEL -o value "$DEVNAME")
  if [ -z "$LABEL" ]; then
    LABEL=$(basename "$DEVNAME")
  fi
  MNT_DIR="/mnt/usb_$LABEL"
  mkdir -p "$MNT_DIR"
  mount -o uid=1000,gid=1000,umask=000 "$DEVNAME" "$MNT_DIR"
fi
EOF
chmod +x ovl_root/usr/local/bin/auto-mount.sh

```

#### 5.3 Early local.d script to install cache packages

```zsh
mkdir -p ovl_root/etc/local.d
cat << 'EOF' > ovl_root/etc/local.d/00-install-cache.start
#!/bin/sh
echo "Installing packages from boot media cache..."

# Try multiple possible mount points for the cache
CACHE_FOUND=0

# First, try common boot media mount points
for MOUNT_POINT in /media/boot /media/ENTROPYLAB /mnt/ENTROPYLAB /boot /media/*; do
  if [ -d "$MOUNT_POINT/cache" ] && [ -n "$(ls -A$MOUNT_POINT/cache/*.apk 2>/dev/null | head -1)" ]; then
    echo "Found cache at: $MOUNT_POINT/cache"
    apk add --allow-untrusted $MOUNT_POINT/cache/*.apk 2>&1 | head -20
    CACHE_FOUND=1
    break
  fi
done

if [ $CACHE_FOUND -eq 0 ]; then
  echo "WARNING: Could not find cache packages. Packages must be installed manually or via network."
else
  echo "Cache package installation completed."
fi
EOF
chmod +x ovl_root/etc/local.d/00-install-cache.start

```

#### 5.4 Copy the app and create the startup service

*(Updated with robust dependency order, hardware group permissions, fallback cache installation, and profile directory protection).*

```zsh
mkdir -p ovl_root/var/www/entropylab
cp -R app_assets/* ovl_root/var/www/entropylab/
chmod -R 755 ovl_root/var/www/entropylab

mkdir -p ovl_root/etc/init.d
cat << 'EOF' > ovl_root/etc/init.d/entropylab
#!/sbin/openrc-run
name="EntropyLab App"

depend() {
  after localmount local eudev local.d
  keyword -jail
}

start_pre() {
  if ! id -u entropylab >/dev/null 2>&1; then
    adduser -D -u 1000 -G video,input -s /bin/ash entropylab
  fi
  chown -R entropylab:entropylab /home/entropylab
}

start() {
  ebegin "Starting Hardened EntropyLab App"

  # Fallback check to install packages from cache if not already present
  if ! command -v httpd >/dev/null 2>&1 || ! command -v cage >/dev/null 2>&1; then
    for MOUNT_POINT in /media/boot /media/ENTROPYLAB /mnt/ENTROPYLAB /boot /media/*; do
      if [ -d "$MOUNT_POINT/cache" ]; then
        apk add --allow-untrusted --no-network $MOUNT_POINT/cache/*.apk 2>/dev/null
        break
      fi
    done
  fi

  export XDG_RUNTIME_DIR=/tmp/runtime-root
  mkdir -p $XDG_RUNTIME_DIR
  chown -R entropylab:entropylab $XDG_RUNTIME_DIR
  chmod 0700 $XDG_RUNTIME_DIR

  # Ensure user-data dir exists and is owned by entropylab to prevent crashes
  mkdir -p /tmp/chrome
  chown -R entropylab:entropylab /tmp/chrome
  chmod 0700 /tmp/chrome

  # Local read-only web server (Same-Origin Policy sandbox)
  httpd -p 127.0.0.1:8080 -h /var/www/entropylab -u nobody:nobody

  su - entropylab -c "
    export XDG_RUNTIME_DIR=/tmp/runtime-root
    cage -d -- chromium-browser \
      --incognito \
      --no-first-run \
      --no-default-browser-check \
      --bwsi \
      --disable-sync \
      --disable-extensions \
      --disable-component-update \
      --disable-component-extensions-with-background-pages \
      --disable-notifications \
      --disable-background-networking \
      --disable-client-side-phishing-detection \
      --disable-session-crashed-bubble \
      --disable-infobars \
      --disable-breakpad \
      --disable-domain-reliability \
      --disable-speech-api \
      --no-pings \
      --disable-features=AccountConsistency,TranslateUI,MediaRouter,DialMediaRouteProvider,AutofillServerCommunication,CertificateTransparencyComponentUpdater,OptimizationHints \
      --password-store=basic \
      --user-data-dir=/tmp/chrome \
      [http://127.0.0.1:8080/entropylab.html](http://127.0.0.1:8080/entropylab.html) &
  "
  eend $?
}

stop() {
  ebegin "Stopping EntropyLab App"
  killall chromium-browser cage httpd
  eend $?
}
EOF
chmod +x ovl_root/etc/init.d/entropylab

```

#### 5.5 Enable services at boot

```zsh
mkdir -p ovl_root/etc/runlevels/{sysinit,default}
echo "entropylab" > ovl_root/etc/hostname

ln -sf /etc/init.d/udev        ovl_root/etc/runlevels/sysinit/udev
ln -sf /etc/init.d/udev-trigger ovl_root/etc/runlevels/sysinit/udev-trigger
ln -sf /etc/init.d/entropylab  ovl_root/etc/runlevels/default/entropylab

```

> The overlay is packaged only after the kernel/modloop work below is finished.

---

### 6. Download Alpine, scrub modloop (drivers + firmware), lock down radios

#### 6.1 Download and extract Alpine Raspberry Pi image (v3.20.10)

```zsh
curl -LO [https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-rpi-3.20.10-aarch64.tar.gz](https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-rpi-3.20.10-aarch64.tar.gz)
tar -xzf alpine-rpi-3.20.10-aarch64.tar.gz -C boot/
rm alpine-rpi-3.20.10-aarch64.tar.gz

```

#### 6.2 Locate modloop and rebuild it (remove drivers + firmware)

```zsh
MODLOOP=$(find boot -name 'modloop-rpi' -type f | head -1)
if [ -z "$MODLOOP" ]; then
  echo "❌ modloop-rpi not found after extraction."
  echo "   Contents of boot/ (first 20 entries):"
  find boot -type f | head -20
  echo "   Check that the Alpine tarball downloaded and extracted correctly."
  exit 1
fi
echo "Found modloop at: $MODLOOP"

# Copy modloop to a temp location for modification (fixes permission issues)
MODLOOP_TEMP="/tmp/modloop-rpi-build"
rm -rf "$MODLOOP_TEMP"
cp "$MODLOOP" "$MODLOOP_TEMP"

docker run --rm -v "$(pwd):/work" -v "$MODLOOP_TEMP:/modloop_temp" -w /work --platform linux/arm64 alpine:latest sh -c "
  apk add --no-cache squashfs-tools && \
  unsquashfs -d /tmp/modloop /modloop_temp && \
  # Remove network / wireless / Bluetooth drivers
  rm -rf \
    /tmp/modloop/modules/*/kernel/drivers/net \
    /tmp/modloop/modules/*/kernel/drivers/bluetooth \
    /tmp/modloop/modules/*/kernel/drivers/net/wireless \
    /tmp/modloop/modules/*/kernel/net && \
  # Remove common wireless / Bluetooth firmware (Broadcom, Cypress, etc.)
  rm -rf \
    /tmp/modloop/modules/firmware/brcm* \
    /tmp/modloop/modules/firmware/cypress* \
    /tmp/modloop/modules/firmware/ath* \
    /tmp/modloop/modules/firmware/iwlwifi* \
    /tmp/modloop/modules/firmware/rtlwifi* \
    /tmp/modloop/modules/firmware/rt* \
    /tmp/modloop/modules/firmware/ti-connectivity \
    /tmp/modloop/modules/firmware/bluetooth && \
  # Clean dependency index files so the kernel does not look for removed modules
  find /tmp/modloop -type f \\( \     -name 'modules.dep*' -o -name 'modules.alias*' -o \     -name 'modules.symbols*' -o -name 'modules.builtin*' -o \     -name 'modules.devname' -o -name 'modules.softdep' \   \\) -delete && \
  mksquashfs /tmp/modloop /modloop_temp -noappend -comp xz
"

# Copy the modified modloop back to the original location
cp "$MODLOOP_TEMP" "$MODLOOP"
rm "$MODLOOP_TEMP"
echo "✅ modloop rebuilt and hardened."

```

#### 6.3 Disable Wi-Fi / Bluetooth radios in firmware config

```zsh
cat << 'EOF' >> boot/usercfg.txt
dtoverlay=disable-wifi
dtoverlay=disable-bt
gpu_mem=128
EOF

```

#### 6.4 Disable kernel IP stack

```zsh
gsed -i 's/$/ ip=off/' boot/cmdline.txt

```

#### 6.5 Package the overlay (must be last step that touches ovl_root)

```zsh
tar -czf boot/entropylab.apkovl.tar.gz -C ovl_root .
echo "✅ Overlay packaged as boot/entropylab.apkovl.tar.gz"

```

---

### 7. Create the bootable media

#### Option A – Flash directly to microSD or USB stick

> **Password note:** `diskutil partitionDisk` will ask for your **macOS User Administrator Password**.

##### 7A.1 List available disks

```zsh
diskutil list

```

**What it does:** Displays all mounted disks. Look for your microSD card or USB stick and note its identifier (e.g., `disk4`).

---

##### 7A.2 Prompt for disk identifier

After reviewing the disk list above, run this to select which disk to erase:

```zsh
read "TARGET_DISK?Enter the disk identifier to erase (example: disk4): "
if [ -z "$TARGET_DISK" ]; then
  echo "❌ No disk selected. Aborting."
  exit 1
fi

```

---

##### 7A.3 Validate the disk exists

```zsh
if ! diskutil info "/dev/$TARGET_DISK" >/dev/null 2>&1; then
  echo "❌ Disk /dev/$TARGET_DISK not found."
  exit 1
fi

```

---

##### 7A.4 Show disk details and confirm erasure

```zsh
DISK_SIZE=$(diskutil info "/dev/$TARGET_DISK" | grep "Disk Size" | sed 's/.*: *//')
echo ""
echo "About to erase: /dev/$TARGET_DISK"
echo "Size: $DISK_SIZE"
read "CONFIRM?This cannot be undone. Type YES to confirm: "
if [ "$CONFIRM" != "YES" ]; then
  echo "❌ Aborted."
  exit 1
fi

```

**What it does:** Shows you the disk size and asks for final confirmation. Type **`YES`** exactly to proceed.

---

##### 7A.5 Erase and partition the disk

```zsh
echo "Erasing and partitioning /dev/$TARGET_DISK ..."
diskutil partitionDisk "/dev/$TARGET_DISK" MBR "MS-DOS FAT32" ENTROPYLAB 0b

```

**Note:** This will prompt for your **macOS admin password**.

---

##### 7A.6 Wait for the volume to mount

```zsh
echo "Waiting for /Volumes/ENTROPYLAB ..."
ATTEMPTS=0
MAX_ATTEMPTS=30
while [ $ATTEMPTS -lt$MAX_ATTEMPTS ]; do
  if [ -d /Volumes/ENTROPYLAB ]; then
    echo "✅ Volume mounted."
    break
  fi
  echo "⏳ Waiting for /Volumes/ENTROPYLAB... ($((ATTEMPTS+1))/$MAX_ATTEMPTS)"
  sleep 1
  ATTEMPTS=$((ATTEMPTS + 1))
done

if [ ! -d /Volumes/ENTROPYLAB ]; then
  echo "❌ Timeout: /Volumes/ENTROPYLAB never appeared."
  echo "   Check Disk Utility. Did the partition step succeed?"
  exit 1
fi

```

---

##### 7A.7 Copy files to the disk

```zsh
echo "Copying files..."
cp -R boot/* /Volumes/ENTROPYLAB/
mkdir -p /Volumes/ENTROPYLAB/cache
cp -R cache/* /Volumes/ENTROPYLAB/cache/

```

---

##### 7A.8 Safely eject the disk

```zsh
sync
diskutil eject /Volumes/ENTROPYLAB
echo "✅ Done. Card/stick is ready to boot on a Raspberry Pi 4/5."

```

---

#### Option B – Create a distributable .img file

```zsh
docker run --rm -v "$(pwd):/work" -w /work --platform linux/arm64 alpine:latest sh -c "
  apk add --no-cache dosfstools mtools && \
  dd if=/dev/zero of=entropylab_rpi.img bs=1M count=320 && \
  mkfs.vfat -F 32 -n ENTROPYLAB entropylab_rpi.img && \
  mcopy -i entropylab_rpi.img -s boot/* ::/ && \
  mcopy -i entropylab_rpi.img -s cache ::/
"

if [ -f entropylab_rpi.img ]; then
  IMG_SIZE=$(ls -lh entropylab_rpi.img | awk '{print $5}')
  echo "✅ Image created: entropylab_rpi.img ($IMG_SIZE)"
  echo "   Quick listing of contents:"
  docker run --rm -v "$(pwd):/work" -w /work --platform linux/arm64 alpine:latest sh -c "
    apk add --no-cache mtools >/dev/null && \
    mdir -i /work/entropylab_rpi.img ::/ | head -20
  "
  echo "   You can flash this later with Raspberry Pi Imager, balenaEtcher, or dd."
else
  echo "❌ Image creation failed — entropylab_rpi.img not found."
  exit 1
fi

```

---

## Quick checklist before you boot the Pi

* [ ] `app_assets/entropylab.html` was present when you ran the build
* [ ] You saw the messages "✅ Overlay packaged" and "✅ Done" / "✅ Image created"
* [ ] The microSD / USB is safely ejected
* [ ] Pi has at least 2 GB RAM
* [ ] No network cable is attached (optional but recommended for the air-gapped use case)

Insert the card, power on the Pi, and EntropyLab should start automatically in full-screen Chromium.
