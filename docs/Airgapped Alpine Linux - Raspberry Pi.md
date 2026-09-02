# Airgapped Alpine Linux Build Guide

**Hardened RAM-only OS for Raspberry Pi 4/5**

This guide builds a bootable drive or image that boots Alpine diskless, then launches **[OogaBoogaX/entropylab `entropylab.html`](https://github.com/OogaBoogaX/entropylab/blob/rock/entropylab.html)** in Chromium on a local Wayland session. Optimized for Apple Silicon (M1–M4) using OrbStack.

The browser is **fullscreen, not kiosk**. Kiosk mode hides Chromium chrome and blocks the file-save dialogs needed to write recovery data to USB. Fullscreen still fills the display; Save As / file pickers keep working.

> [!CAUTION]
> **Hardware:** Raspberry Pi 4 or 5 with **at least 2 GB of RAM**. The OS and Chromium load into tmpfs. 1 GB boards will OOM.
>
> **Version pin:** Use Alpine **3.24.1** (branch `v3.24`), the current stable as of 2026-06. Do **not** use `alpine:latest` (it drifts, often to edge) and do **not** use 3.20. v3.20 community support ended 2026-04-01. Chromium lives in *community*, so the RPi tarball, Docker image, and apk repos must all be the same branch.

## What the Pi actually needs

EntropyLab’s shippable artifact is **one self-contained HTML file**. The build (`scripts/build.mjs`) inlines CSS, JS, WASM, both header logos (SVG markup), and the favicon (PNG as base64 + SVG). It does **not** load `assets/`, CDNs, fonts.googleapis, or any other file at runtime.

**Copy only `entropylab.html` onto the image.** Do not copy `assets/`, `src/`, WASM crates, or `node_modules`. Those exist for the hosted site and for developers.

PNG/SVG logos in `OogaBoogaX/entropylab/assets/` are for GitHub Pages / social cards. They are already baked into the HTML. Putting them next to the HTML on the Pi does nothing unless you change the app to use relative URLs like `assets/logo.svg` — in that case yes, keep the same relative path next to the HTML so busybox httpd can serve it. For the official build, skip them.

Fonts: DejaVu is the system fallback. Chromium also ships its own UI fonts. No webfont packages.

OS packages (top-level only; `apk fetch --recursive` pulls the rest):

| Package | Why |
|---|---|
| `chromium` | The app. Pulls GTK, NSS, ICU, Wayland/Ozone deps. |
| `mesa-dri-gallium` | Pi GPU. Without this Chromium is often a black screen. |
| `labwc` | Stacking compositor so file-save dialogs work (cage cannot). |
| `seatd` + `seatd-openrc` | Seat for labwc/Chromium. |
| `dbus` + `dbus-openrc` | Chromium/GTK file dialogs. |
| `eudev` + `eudev-openrc` | USB hotplug. |
| `busybox-extras` | `httpd` on 127.0.0.1. |
| `font-dejavu` | Fallback glyphs. |
| `util-linux` | `blkid` for automount. |
| `dosfstools` `exfatprogs` `ntfs-3g` | USB filesystems the user saves onto. |

Not fetched: `alpine-base` / `openrc` (already in the alpine-rpi tarball), `wayland-protocols`, extra mesa bits, `gtk+3.0`, `icu-data-full` — those come in as chromium/labwc dependencies.

---

## Build instructions (macOS Apple Silicon)

### 1. System bootstrap

```zsh
if [ "$(uname -m)" != "arm64" ]; then
    echo "This workflow requires an Apple Silicon Mac."
    exit 1
fi

if ! command -v brew &> /dev/null; then
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
fi

brew update
brew install --cask orbstack
brew install coreutils gnu-sed
export PATH="/opt/homebrew/bin:$PATH"
```

Open **OrbStack** and wait until the engine is Running before any `docker` step.

### 2. Workspace and the HTML artifact

```zsh
mkdir -p ~/entropylab/{boot,cache,ovl_root,app_assets}
cd ~/entropylab

export ALPINE_VERSION=3.24.1
export ALPINE_BRANCH=v3.24

# Official self-contained app (logos/fonts/JS/WASM already inside this file).
curl -fL -o app_assets/entropylab.html \
  https://raw.githubusercontent.com/OogaBoogaX/entropylab/rock/entropylab.html
```

Prefer a release asset you have SHA256-checked (`SHA256SUMS.txt` next to the file in that repo) over `raw.githubusercontent.com` if you are building for real funds.

> [!CAUTION]
> `app_assets/` should contain **only** `entropylab.html`. Do not dump the git tree or `assets/` in here.

### 3. Fetch packages (community repo + verify)

```zsh
docker run --rm -v "$(pwd)":/work -w /work --platform linux/arm64 alpine:${ALPINE_VERSION} sh -c "
  set -euo pipefail
  printf '%s\n' \
    https://dl-cdn.alpinelinux.org/alpine/${ALPINE_BRANCH}/main \
    https://dl-cdn.alpinelinux.org/alpine/${ALPINE_BRANCH}/community \
    > /etc/apk/repositories
  apk update
  mkdir -p /work/cache
  apk fetch --recursive -o /work/cache --no-cache \
    busybox-extras \
    eudev eudev-openrc \
    dbus dbus-openrc \
    seatd seatd-openrc \
    labwc \
    chromium \
    mesa-dri-gallium \
    font-dejavu \
    util-linux \
    dosfstools exfatprogs ntfs-3g
  cd /work/cache
  apk verify ./*.apk
  apk index -o APKINDEX.tar.gz ./*.apk
  ls -lh
  du -sh .
"
```

### 4. Overlay (apkovl)

#### 4.1 User exists before udev (no chicken-and-egg)

Do **not** create the user only in the default-runlevel app service. USB automount is sysinit. Create uid 1000 in a sysinit service that runs **before** udev. Mount options use numeric `uid=1000` so VFAT/exFAT work even if name lookup lags.

```zsh
mkdir -p ovl_root/home/entropylab/.config/labwc
mkdir -p ovl_root/etc/init.d ovl_root/etc/local.d
mkdir -p ovl_root/etc/modprobe.d ovl_root/etc/apk
mkdir -p ovl_root/etc/runlevels/{sysinit,boot,default}
mkdir -p ovl_root/usr/local/bin ovl_root/etc/udev/rules.d

cat << 'EOF' > ovl_root/etc/init.d/entropylab-prep
#!/sbin/openrc-run
name="EntropyLab prep"

depend() {
    before udev udev-trigger localmount
    keyword -jail -lxc -docker
}

start() {
    ebegin "Preparing entropylab user"
    if ! getent group seat >/dev/null 2>&1; then
        addgroup -S seat
    fi
    if ! id -u entropylab >/dev/null 2>&1; then
        adduser -D -u 1000 -s /bin/ash entropylab
    fi
    for g in video input audio seat; do
        addgroup entropylab "$g" 2>/dev/null || true
    done
    mkdir -p /home/entropylab
    chown -R 1000:1000 /home/entropylab
    eend 0
}
EOF
chmod +x ovl_root/etc/init.d/entropylab-prep
```

#### 4.2 USB / SD automount (numeric uid, safe labels, fstype-aware)

```zsh
cat << 'EOF' > ovl_root/etc/udev/rules.d/99-automount.rules
ACTION=="add", SUBSYSTEM=="block", ENV{DEVTYPE}=="partition", RUN+="/usr/local/bin/auto-mount.sh %N"
EOF

cat << 'EOF' > ovl_root/usr/local/bin/auto-mount.sh
#!/bin/sh
DEVNAME=$1
[ -z "$DEVNAME" ] && exit 0
case "$DEVNAME" in
    /dev/*) ;;
    *) DEVNAME="/dev/$DEVNAME" ;;
esac

# Skip the boot volume (already mounted).
if mount | grep -q " ${DEVNAME} "; then
    exit 0
fi

BUS=$(udevadm info --query=property --name="$DEVNAME" 2>/dev/null | sed -n 's/^ID_BUS=//p' | head -n1)
FSTYPE=$(blkid -s TYPE -o value "$DEVNAME" 2>/dev/null)
LABEL=$(blkid -s LABEL -o value "$DEVNAME" 2>/dev/null)
LABEL=$(printf '%s' "$LABEL" | tr -c 'A-Za-z0-9._-' '_')
[ -z "$LABEL" ] && LABEL=$(basename "$DEVNAME")

if [ "$BUS" = "mmc" ]; then
    MNT_DIR=/mnt/sdcard
else
    MNT_DIR="/mnt/usb_$LABEL"
fi

mkdir -p "$MNT_DIR"

mount_fat_like() {
    mount -o uid=1000,gid=1000,umask=000,noexec,nodev,nosuid "$DEVNAME" "$MNT_DIR"
}

case "$FSTYPE" in
    vfat|msdos|fat|exfat|ntfs|ntfs-3g)
        mount_fat_like || mount "$DEVNAME" "$MNT_DIR"
        ;;
    *)
        if ! mount "$DEVNAME" "$MNT_DIR"; then
            mount_fat_like || exit 0
        fi
        chown -R 1000:1000 "$MNT_DIR" 2>/dev/null || chmod 0777 "$MNT_DIR"
        ;;
esac
EOF
chmod +x ovl_root/usr/local/bin/auto-mount.sh
```

`noexec` on removable media is extra hardening; EntropyLab only needs to **write files**, not run binaries from USB.

#### 4.3 Install cached packages on first boot

World lists only the top-level set. Recursive `.apk` files in `cache/` still get installed because the local.d script adds every file in that directory.

```zsh
cat << 'EOF' > ovl_root/etc/apk/world
busybox-extras
eudev
eudev-openrc
dbus
dbus-openrc
seatd
seatd-openrc
labwc
chromium
mesa-dri-gallium
font-dejavu
util-linux
dosfstools
exfatprogs
ntfs-3g
EOF

cat << 'EOF' > ovl_root/etc/local.d/00-apk-from-cache.start
#!/bin/sh
CACHE=""
for d in /media/*/cache /cache; do
    if [ -d "$d" ] && ls "$d"/*.apk >/dev/null 2>&1; then
        CACHE=$d
        break
    fi
done
[ -z "$CACHE" ] && exit 0

printf '%s\n' "$CACHE" > /etc/apk/repositories
ln -sfn "$CACHE" /etc/apk/cache

if ! command -v chromium-browser >/dev/null 2>&1; then
    apk add --no-network --allow-untrusted --force-non-repository $(ls "$CACHE"/*.apk)
fi
exit 0
EOF
chmod +x ovl_root/etc/local.d/00-apk-from-cache.start
```

`--allow-untrusted` is required because the local `APKINDEX.tar.gz` is not signed. Packages were already `apk verify`'d when fetched.

#### 4.4 labwc + Chromium (fullscreen, not kiosk)

```zsh
mkdir -p ovl_root/var/www/entropylab
cp -R app_assets/. ovl_root/var/www/entropylab/
chmod -R 755 ovl_root/var/www/entropylab
# Expected: ovl_root/var/www/entropylab/entropylab.html only.

cat << 'EOF' > ovl_root/home/entropylab/.config/labwc/autostart
chromium-browser \
  --ozone-platform=wayland \
  --ozone-platform-hint=wayland \
  --enable-features=UseOzonePlatform,WaylandWindowDecorations \
  --start-fullscreen \
  --incognito \
  --no-first-run \
  --disable-sync \
  --disable-extensions \
  --disable-component-update \
  --disable-notifications \
  --disable-infobars \
  --noerrdialogs \
  --disable-session-crashed-bubble \
  --disable-features=TranslateUI \
  --disable-dev-shm-usage \
  --user-data-dir=/tmp/chrome \
  http://127.0.0.1:8080/entropylab.html &
EOF
# --disable-gpu-compositing
# Uncomment the flag above (add it to the chromium-browser invocation) only if the Pi compositor is blank or flickering.

cat << 'EOF' > ovl_root/etc/init.d/entropylab
#!/sbin/openrc-run
name="EntropyLab App"
command="/usr/bin/labwc"
command_user="entropylab"

depend() {
    need local dbus seatd
    after localmount udev-trigger entropylab-prep
    keyword -jail
}

start_pre() {
    if ! command -v chromium-browser >/dev/null 2>&1; then
        /etc/local.d/00-apk-from-cache.start || true
    fi
    if ! id -u entropylab >/dev/null 2>&1; then
        adduser -D -u 1000 -s /bin/ash entropylab
    fi
    mkdir -p /tmp/runtime-entropylab /tmp/chrome
    chown -R entropylab:entropylab /tmp/runtime-entropylab /tmp/chrome /home/entropylab
    chmod 0700 /tmp/runtime-entropylab
}

start() {
    ebegin "Starting EntropyLab (labwc + Chromium fullscreen)"
    httpd -p 127.0.0.1:8080 -h /var/www/entropylab -u nobody:nobody

    export XDG_RUNTIME_DIR=/tmp/runtime-entropylab
    export XDG_SESSION_TYPE=wayland
    su - entropylab -c "
      export XDG_RUNTIME_DIR=/tmp/runtime-entropylab
      export XDG_SESSION_TYPE=wayland
      exec labwc
    " &
    eend $?
}

stop() {
    ebegin "Stopping EntropyLab"
    killall chromium-browser labwc httpd 2>/dev/null || true
    eend 0
}
EOF
chmod +x ovl_root/etc/init.d/entropylab
```

#### 4.5 Net-driver blacklist (this one actually works)

```zsh
cat << 'EOF' > ovl_root/etc/modprobe.d/airgap-net.conf
blacklist brcmfmac
blacklist brcmutil
blacklist hci_uart
blacklist btbcm
blacklist bluetooth
blacklist cfg80211
blacklist rfkill
install brcmfmac /bin/true
install brcmutil /bin/true
install bluetooth /bin/true
install cfg80211 /bin/true
EOF
```

#### 4.6 Enable services and pack overlay

```zsh
echo "entropylab" > ovl_root/etc/hostname

ln -sf /etc/init.d/entropylab-prep ovl_root/etc/runlevels/sysinit/entropylab-prep
ln -sf /etc/init.d/udev            ovl_root/etc/runlevels/sysinit/udev
ln -sf /etc/init.d/udev-trigger    ovl_root/etc/runlevels/sysinit/udev-trigger
ln -sf /etc/init.d/local           ovl_root/etc/runlevels/default/local
ln -sf /etc/init.d/dbus            ovl_root/etc/runlevels/default/dbus
ln -sf /etc/init.d/seatd           ovl_root/etc/runlevels/default/seatd
ln -sf /etc/init.d/entropylab      ovl_root/etc/runlevels/default/entropylab

tar -czf boot/localhost.apkovl.tar.gz -C ovl_root .
```

`localhost.apkovl.tar.gz` is the name the initramfs looks for before hostname is applied. Keep it.

### 5. Kernel, firmware, cmdline

```zsh
curl -fLO "https://dl-cdn.alpinelinux.org/alpine/${ALPINE_BRANCH}/releases/aarch64/alpine-rpi-${ALPINE_VERSION}-aarch64.tar.gz"
tar -xzf "alpine-rpi-${ALPINE_VERSION}-aarch64.tar.gz" -C boot/
rm "alpine-rpi-${ALPINE_VERSION}-aarch64.tar.gz"

cat << 'EOF' >> boot/usercfg.txt
dtoverlay=disable-wifi
dtoverlay=disable-bt
gpu_mem=128
EOF

gsed -i 's/$/ ip=off ipv6.disable=1 modprobe.blacklist=brcmfmac,brcmutil,hci_uart,btbcm,bluetooth,cfg80211,rfkill/' boot/cmdline.txt
```

### 6. Distribution

Chromium + Mesa + Wayland is hundreds of MB of `.apk` files. Size the FAT image from actual contents plus 25% headroom, minimum 2048 MB.

**Option A: flash microSD / USB**

```zsh
diskutil list
# Replace <disk_id> with the target (e.g. disk4). This WIPEs the disk.
diskutil partitionDisk /dev/<disk_id> MBR "MS-DOS FAT32" ENTROPYLAB 0b
cp -R boot/. /Volumes/ENTROPYLAB/
mkdir -p /Volumes/ENTROPYLAB/cache
cp -R cache/. /Volumes/ENTROPYLAB/cache/
sync
diskutil eject /Volumes/ENTROPYLAB
```

**Option B: distributable .img (auto-sized)**

```zsh
NEED_MB=$(du -sm boot cache | awk '{s+=$1} END {print s}')
IMG_MB=$(( NEED_MB + NEED_MB / 4 + 256 ))
if [ "$IMG_MB" -lt 2048 ]; then IMG_MB=2048; fi
echo "Building ${IMG_MB} MB image (payload ${NEED_MB} MB)"

docker run --rm -v "$(pwd)":/work -w /work --platform linux/arm64 alpine:${ALPINE_VERSION} sh -c "
  set -euo pipefail
  apk add --no-cache dosfstools mtools
  dd if=/dev/zero of=entropylab_rpi.img bs=1M count=${IMG_MB} status=none
  mkfs.vfat -F 32 -n ENTROPYLAB entropylab_rpi.img
  mcopy -i entropylab_rpi.img -s boot/* ::/
  mcopy -i entropylab_rpi.img -s cache ::/
"
ls -lh entropylab_rpi.img
echo "Distributable image: entropylab_rpi.img"
```

Flash the `.img` with Raspberry Pi Imager or `dd`. First boot installs packages from `cache/` into RAM; that takes a minute.

### Verify after first boot (on the Pi, if you attach a keyboard)

```sh
apk policy chromium labwc
command -v chromium-browser
ls /var/www/entropylab
ls /mnt
cat /proc/cmdline
lsmod | grep -E 'brcmfmac|bluetooth' || echo "radio modules not loaded"
```

Save from EntropyLab to `/mnt/usb_<LABEL>/` (shown in Chromium as that path in the file picker).
