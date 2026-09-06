# Airgapped Alpine Linux Build Guide

**RAM-only Alpine for Raspberry Pi 4 / 5 — built entirely on macOS**

This guide produces a FAT32 SD card that boots Alpine Linux into RAM and lands on a minimal desktop as user `entropylab`, with Chromium already installed. The Pi never brings up Ethernet, Wi-Fi, or Bluetooth. You drop `entropylab.html` onto the card on your Mac **just before ejecting**.

Optimized for **Apple Silicon Macs** (M1–M4) running **macOS Tahoe**, using **OrbStack** (or Docker Desktop).

> **Hardware:** Raspberry Pi 4 or 5 with **4 GB RAM or more**. The whole OS plus Chromium lives in RAM. 2 GB models will likely fail or thrash.

---

## What this build is (and is not)

| Item | Choice |
| --- | --- |
| Alpine | `latest-stable`, aarch64 |
| Mode | Diskless — entire root filesystem in tmpfs (RAM) |
| Display | Xorg + Openbox desktop (panel, file manager, terminal) |
| Browser | Chromium, started normally from the desktop menu (not kiosk) |
| HTML | Pure `file://` from a `html/` folder you add on the Mac |
| User | `entropylab`, auto-login, password `entropylab` |
| Network on the Pi | Never started. Wi-Fi and Bluetooth disabled in firmware |
| Persistence | None. Normal power-off is enough |
| Output | Folder of files copied onto a FAT32 SD card |

**Left out on purpose:** SSH, Wi-Fi tools, Bluetooth, NTP, extra editors, local web server, kiosk/fullscreen mode, offline package re-install after the first build.

### How boot actually works (read this once)

Alpine **diskless** still “installs” packages **every boot**. That is not a setup session and it is **not** using the network.

1. Pi firmware reads the FAT32 card (kernel, initramfs, firmware).
2. Initramfs creates a tmpfs root in RAM.
3. A boot script installs the `.apk` files from `cache/` **on the card** into that RAM disk.
4. The pre-built overlay (`entropylab.apkovl.tar.gz`) is applied (user, auto-login, X, Chromium flags).
5. Auto-login as `entropylab` → `startx` → Openbox desktop.
6. `html/` is copied into `/tmp/html` (RAM). The boot partition is remounted **read-only**.
7. You open Chromium and load `file:///tmp/html/entropylab.html`.
8. Power off. RAM is gone. The card still only has what you copied onto it.

Every boot can take **1–2 minutes** while Chromium unpacks into RAM. That is normal.

The Pi clock will be wrong (no battery RTC, no NTP). Local HTML / BIP39 does not need correct time.

---

## Requirements

**Mac**

- Apple Silicon Mac, macOS Tahoe
- OrbStack (preferred) or Docker Desktop
- ~3 GB free disk for the build
- An SD card you are willing to erase (8 GB+, 16 GB is comfortable)

**Pi**

- Raspberry Pi 4 or 5 (aarch64)
- 4 GB+ RAM
- HDMI display, USB keyboard, USB mouse
- **No Ethernet cable**

**You supply last**

- `entropylab.html` (copied into `html/` on the card just before you take it to the Pi)

---

## 1. Create a working folder

Create a folder without spaces, for example `EntropyLab-Build`.

Right-click it → **New Terminal at Folder** (or `cd` into it).

All commands below assume you are already in that folder.

---

## 2. One-time Mac bootstrap

> Homebrew / OrbStack may ask for your **macOS admin password**.

```zsh
# Must be Apple Silicon
if [ "$(uname -m)" != "arm64" ]; then
  echo "This workflow requires an Apple Silicon Mac."
  exit 1
fi

if [[ "$PWD" == *" "* ]]; then
  echo "Warning: this folder path contains spaces. A path without spaces is simpler."
fi

if ! command -v brew >/dev/null 2>&1; then
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
fi

export PATH="/opt/homebrew/bin:$PATH"

brew update
brew install --cask orbstack
brew install coreutils gnu-sed

echo "Bootstrap complete."
```

Open **OrbStack** from Applications once so the Docker engine starts.

Check:

```zsh
docker version
docker run --rm --platform linux/arm64 alpine:latest uname -m
```

You want `aarch64`.

---

## 3. Create folders

```zsh
mkdir -p boot cache ovl_root html
```

Do **not** put the HTML in yet. That happens on the SD card, last.

---

## 4. Download packages into `cache/` (Mac uses the network; the Pi will not)

```zsh
if ! command -v docker >/dev/null 2>&1; then
  echo "Docker not found. Open OrbStack, then re-run this step."
  exit 1
fi

open -a OrbStack 2>/dev/null || true

RETRY=0
until docker info >/dev/null 2>&1; do
  if [ "$RETRY" -ge 30 ]; then
    echo "Docker engine did not become ready. Open OrbStack and try again."
    exit 1
  fi
  echo "Waiting for Docker... ($((RETRY+1))/30)"
  sleep 1
  RETRY=$((RETRY + 1))
done

docker run --rm -v "$(pwd):/work" -w /work --platform linux/arm64 alpine:latest sh -c '
  echo "https://dl-cdn.alpinelinux.org/alpine/latest-stable/main" > /etc/apk/repositories
  echo "https://dl-cdn.alpinelinux.org/alpine/latest-stable/community" >> /etc/apk/repositories
  apk update
  apk fetch --recursive -o /work/cache \
    eudev eudev-openrc \
    dbus dbus-openrc \
    agetty shadow \
    xorg-server xinit xset xsetroot \
    xf86-input-libinput \
    xf86-video-fbdev \
    xf86-video-modesetting \
    mesa-dri-gallium mesa-egl mesa-gl \
    setxkbmap \
    font-dejavu \
    hicolor-icon-theme \
    openbox \
    tint2 \
    pcmanfm \
    xterm \
    chromium
'

echo "Cache packages:"
ls cache | wc -l
```

You should see a large number of `.apk` files (Chromium pulls in a lot of dependencies). That is expected.

---

## 5. Build the overlay (apkovl)

This overlay is applied on every boot. It does **not** contain `entropylab.html`.

**Important:** Paste **one block at a time**. Wait for `Block X OK` (or `Overlay tree ready.`) before the next block. A single giant paste is what caused the earlier `zsh: parse error near ')'`.

If you already tried step 5 and it failed, start clean:

```zsh
rm -rf ovl_root
mkdir -p ovl_root
```

### Block A — folders + no-network

```zsh
mkdir -p ovl_root/etc/network \
         ovl_root/etc/runlevels/sysinit \
         ovl_root/etc/runlevels/default \
         ovl_root/etc/local.d \
         ovl_root/etc/X11/xorg.conf.d \
         ovl_root/etc/chromium \
         ovl_root/home/entropylab/.config/openbox \
         ovl_root/usr/local/bin

echo "entropylab" > ovl_root/etc/hostname

cat > ovl_root/etc/network/interfaces << 'EOF'
auto lo
iface lo inet loopback
EOF

ln -sf /etc/init.d/udev         ovl_root/etc/runlevels/sysinit/udev
ln -sf /etc/init.d/udev-trigger ovl_root/etc/runlevels/sysinit/udev-trigger
ln -sf /etc/init.d/dbus         ovl_root/etc/runlevels/default/dbus
ln -sf /etc/init.d/local        ovl_root/etc/runlevels/default/local

echo "Block A OK"
```

### Block B — boot scripts

```zsh
cat > ovl_root/etc/local.d/00-install-cache.start << 'EOF'
#!/bin/sh
echo "Installing packages from boot media cache into RAM..."
CACHE_FOUND=0
for MOUNT_POINT in /media/mmcblk0p1 /media/ENTROPYLAB /media/ALPINE /media/boot /media/*; do
  if [ -d "$MOUNT_POINT/cache" ] && ls "$MOUNT_POINT/cache"/*.apk >/dev/null 2>&1; then
    echo "Found cache at: $MOUNT_POINT/cache"
    apk add --allow-untrusted --no-network "$MOUNT_POINT/cache"/*.apk
    CACHE_FOUND=1
    break
  fi
done
if [ "$CACHE_FOUND" -eq 0 ]; then
  echo "WARNING: cache/*.apk not found on boot media."
fi
EOF
chmod +x ovl_root/etc/local.d/00-install-cache.start

cat > ovl_root/usr/local/bin/entropylab-firstboot.sh << 'EOF'
#!/bin/sh
addgroup -S video 2>/dev/null || true
addgroup -S input 2>/dev/null || true
addgroup -S audio 2>/dev/null || true
if ! id entropylab >/dev/null 2>&1; then
  adduser -D -s /bin/ash entropylab
  echo "entropylab:entropylab" | chpasswd
  addgroup entropylab video 2>/dev/null || true
  addgroup entropylab input 2>/dev/null || true
  addgroup entropylab audio 2>/dev/null || true
fi
chown -R entropylab:entropylab /home/entropylab 2>/dev/null || true
exit 0
EOF
chmod +x ovl_root/usr/local/bin/entropylab-firstboot.sh

cat > ovl_root/etc/local.d/05-user.start << 'EOF'
#!/bin/sh
/usr/local/bin/entropylab-firstboot.sh
EOF
chmod +x ovl_root/etc/local.d/05-user.start

cat > ovl_root/etc/local.d/10-html-copy.start << 'EOF'
#!/bin/sh
boot=""
for d in /media/mmcblk0p1 /media/ENTROPYLAB /media/ALPINE /media/*; do
  [ -d "$d" ] || continue
  if [ -f "$d/cmdline.txt" ] || [ -d "$d/html" ]; then
    boot="$d"
    break
  fi
done
mkdir -p /tmp/html
chmod 755 /tmp/html
if [ -n "$boot" ] && [ -d "$boot/html" ]; then
  cp -a "$boot/html/." /tmp/html/ 2>/dev/null || true
  chmod -R a+rX /tmp/html
fi
if [ -n "$boot" ]; then
  mount -o remount,ro "$boot" 2>/dev/null || true
fi
exit 0
EOF
chmod +x ovl_root/etc/local.d/10-html-copy.start

echo "Block B OK"
```

### Block C — Xorg + Chromium

```zsh
cat > ovl_root/etc/X11/xorg.conf.d/99-vc4.conf << 'EOF'
Section "OutputClass"
    Identifier "vc4"
    MatchDriver "vc4"
    Driver "modesetting"
    Option "PrimaryGPU" "true"
EndSection
EOF

cat > ovl_root/etc/chromium/chromium.conf << 'EOF'
CHROMIUM_FLAGS="--no-sandbox --disable-gpu-sandbox --user-data-dir=/tmp/chromium-data --no-first-run --no-default-browser-check --disable-sync --disable-background-networking --disable-features=TranslateUI"
EOF

echo "Block C OK"
```

### Block D — user session

Uses `` `tty` `` instead of `$(tty)` so a bad paste is less likely to confuse zsh.

```zsh
cat > ovl_root/home/entropylab/.profile << 'EOF'
# Wait until X is installed, then start it on tty1.
if [ -z "$DISPLAY" ]; then
  case `tty` in
    /dev/tty1)
      i=0
      while [ "$i" -lt 120 ]; do
        command -v startx >/dev/null 2>&1 && break
        sleep 2
        i=`expr "$i" + 1`
      done
      exec startx
      ;;
  esac
fi
EOF

cat > ovl_root/home/entropylab/.xinitrc << 'EOF'
#!/bin/sh
export XDG_RUNTIME_DIR=/tmp/runtime-entropylab
mkdir -p "$XDG_RUNTIME_DIR" /tmp/chromium-data
chmod 700 "$XDG_RUNTIME_DIR"
xset -dpms 2>/dev/null || true
xset s off 2>/dev/null || true
setxkbmap us 2>/dev/null || true
tint2 &
exec openbox-session
EOF
chmod +x ovl_root/home/entropylab/.xinitrc

echo "Block D OK"
```

### Block E — Openbox menu / config

```zsh
cat > ovl_root/home/entropylab/.config/openbox/menu.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<openbox_menu>
<menu id="root-menu" label="EntropyLab">
  <item label="Chromium"><action name="Execute"><execute>chromium-browser</execute></action></item>
  <item label="Open EntropyLab HTML"><action name="Execute"><execute>chromium-browser file:///tmp/html/entropylab.html</execute></action></item>
  <item label="Files"><action name="Execute"><execute>pcmanfm /tmp/html</execute></action></item>
  <item label="Terminal"><action name="Execute"><execute>xterm</execute></action></item>
  <separator />
  <item label="Restart X"><action name="Exit"/></item>
</menu>
</openbox_menu>
EOF

cat > ovl_root/home/entropylab/.config/openbox/rc.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<openbox_config>
  <focus><focusNew>yes</focusNew></focus>
  <theme><name>Clearlooks</name><titleLayout>NLIMC</titleLayout></theme>
  <desktops><number>1</number><firstdesk>1</firstdesk></desktops>
  <keyboard>
    <keybind key="A-F4"><action name="Close"/></keybind>
    <keybind key="A-Tab"><action name="NextWindow"/></keybind>
    <keybind key="W-space"><action name="ShowMenu"><menu>root-menu</menu></action></keybind>
  </keyboard>
  <mouse>
    <context name="Root">
      <mousebind button="Right" action="Press">
        <action name="ShowMenu"><menu>root-menu</menu></action>
      </mousebind>
    </context>
  </mouse>
  <menu><file>menu.xml</file></menu>
</openbox_config>
EOF

echo "Block E OK"
```

### Block F — autologin

```zsh
cat > ovl_root/etc/inittab << 'EOF'
::sysinit:/sbin/openrc sysinit
::sysinit:/sbin/openrc boot
::wait:/sbin/openrc default

tty1::respawn:/sbin/agetty --autologin entropylab --noclear tty1 linux
tty2::respawn:/sbin/getty 38400 tty2
tty3::respawn:/sbin/getty 38400 tty3

::ctrlaltdel:/sbin/reboot
::shutdown:/sbin/openrc shutdown
EOF

echo "Overlay tree ready."
```

### Verify overlay before continuing

```zsh
for f in \
  ovl_root/etc/hostname \
  ovl_root/etc/network/interfaces \
  ovl_root/etc/inittab \
  ovl_root/etc/local.d/00-install-cache.start \
  ovl_root/etc/local.d/05-user.start \
  ovl_root/etc/local.d/10-html-copy.start \
  ovl_root/etc/chromium/chromium.conf \
  ovl_root/home/entropylab/.profile \
  ovl_root/home/entropylab/.xinitrc \
  ovl_root/home/entropylab/.config/openbox/menu.xml \
  ovl_root/home/entropylab/.config/openbox/rc.xml
do
  if [ -e "$f" ]; then echo "OK  $f"; else echo "MISSING  $f"; fi
done
```

Every line should say `OK`. If any say `MISSING`, re-run that block only.

---

## 6. Download Alpine Raspberry Pi boot files (`latest-stable`)

```zsh
RPI_FILE=$(docker run --rm --platform linux/arm64 alpine:latest sh -c '
  wget -qO- https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/aarch64/latest-releases.yaml \
    | awk "\$0 ~ /flavor: alpine-rpi/ {inr=1} inr && \$1 == \"file:\" && \$2 ~ /\\.tar\\.gz$/ {print \$2; exit}"
')

if [ -z "$RPI_FILE" ]; then
  echo "Could not resolve alpine-rpi tarball name from latest-stable."
  exit 1
fi

echo "Downloading $RPI_FILE ..."
curl -fL --progress-bar -o rpi.tar.gz \
  "https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/aarch64/$RPI_FILE"

tar -xzf rpi.tar.gz -C boot/
rm rpi.tar.gz
echo "Boot files extracted."
```

---

## 7. Lock down firmware and kernel networking

```zsh
cat > boot/usercfg.txt << 'EOF'
# EntropyLab air-gapped Pi 4/5
dtoverlay=disable-wifi
dtoverlay=disable-bt

# DRM/KMS for Xorg + Chromium
dtoverlay=vc4-kms-v3d
max_framebuffers=2
disable_fw_kms_setup=1

hdmi_force_hotplug=1
EOF

# Do not let the kernel bring up an IP stack
if ! grep -q 'ip=off' boot/cmdline.txt; then
  gsed -i 's/$/ ip=off/' boot/cmdline.txt
fi

# HDMI console
if ! grep -q 'console=tty1' boot/cmdline.txt; then
  gsed -i 's/$/ console=tty1/' boot/cmdline.txt
fi
```

---

## 8. Package the overlay onto the boot files

```zsh
tar -czf boot/entropylab.apkovl.tar.gz -C ovl_root .
mkdir -p boot/html
cat > boot/html/README.txt << 'EOF'
Put entropylab.html in this folder on your Mac,
then eject the card and boot the Pi.

On the Pi it is copied into RAM as:
  file:///tmp/html/entropylab.html

Right-click the desktop → Open EntropyLab HTML
EOF

echo "Overlay packaged: boot/entropylab.apkovl.tar.gz"
```

---

## 9. Copy onto an SD card (erases the card)

> `diskutil` will ask for your **macOS admin password**.

### 9.1 List disks

```zsh
diskutil list
```

Note the SD card identifier (example: `disk4`). It must **not** be `disk0`.

### 9.2 Select, confirm, erase as FAT32 / MBR

The Pi firmware wants **MBR**, not GPT.

```zsh
read "TARGET_DISK?Enter the disk identifier to erase (example: disk4): "
if [ -z "$TARGET_DISK" ]; then
  echo "No disk selected. Aborting."
  exit 1
fi

if ! diskutil info "/dev/$TARGET_DISK" >/dev/null 2>&1; then
  echo "Disk /dev/$TARGET_DISK not found."
  exit 1
fi

DISK_SIZE=$(diskutil info "/dev/$TARGET_DISK" | grep "Disk Size" | sed 's/.*: *//')
echo ""
echo "About to erase: /dev/$TARGET_DISK"
echo "Size: $DISK_SIZE"
read "CONFIRM?This cannot be undone. Type YES to confirm: "
if [ "$CONFIRM" != "YES" ]; then
  echo "Aborted."
  exit 1
fi

diskutil partitionDisk "/dev/$TARGET_DISK" MBR "MS-DOS FAT32" ENTROPYLAB 0b
```

### 9.3 Wait for `/Volumes/ENTROPYLAB`, copy files

```zsh
ATTEMPTS=0
while [ "$ATTEMPTS" -lt 30 ]; do
  [ -d /Volumes/ENTROPYLAB ] && break
  echo "Waiting for /Volumes/ENTROPYLAB... ($((ATTEMPTS+1))/30)"
  sleep 1
  ATTEMPTS=$((ATTEMPTS + 1))
done

if [ ! -d /Volumes/ENTROPYLAB ]; then
  echo "Timeout: volume never appeared. Check Disk Utility."
  exit 1
fi

echo "Copying boot files..."
ditto boot/ /Volumes/ENTROPYLAB/
mkdir -p /Volumes/ENTROPYLAB/cache
ditto cache/ /Volumes/ENTROPYLAB/cache/
mkdir -p /Volumes/ENTROPYLAB/html

echo "Card is populated. Do NOT eject yet if you still need to add the HTML."
```

You should see at least: `cmdline.txt`, `config.txt`, `usercfg.txt`, `entropylab.apkovl.tar.gz`, `cache/`, `html/`.

---

## 10. Add the HTML (last thing before you walk to the Pi)

With the card still mounted:

```zsh
cp /path/to/entropylab.html /Volumes/ENTROPYLAB/html/entropylab.html
# optional extra files:
# cp other-assets... /Volumes/ENTROPYLAB/html/

sync
diskutil eject /Volumes/ENTROPYLAB
echo "Ejected. Card is ready."
```

Replace `/path/to/entropylab.html` with the real path. You can repeat this any time: mount the card on the Mac, replace the HTML, eject. The Pi never writes it back.

---

## 11. Use on the Pi

1. No Ethernet cable.
2. HDMI + keyboard + mouse.
3. Insert the card, power on.
4. Wait. **1–2 minutes** of text while packages load into RAM is expected.
5. You should get the Openbox desktop, logged in as `entropylab`.
6. **Right-click the desktop** → **Open EntropyLab HTML**  
   (or open Chromium and go to `file:///tmp/html/entropylab.html`).
7. Use Chromium normally (windowed, not kiosk).
8. Power off when done. Nothing useful is written back to the card.

**Passwords** (rarely needed because of auto-login): user `entropylab` / `entropylab`. Root is whatever Alpine left as default unless you set one; you should not need root.

**If Chromium is blank or crashy:** `--no-sandbox` is already set. That is required on musl/Alpine more often than not.

**If there is no HTML:** the file was not named `entropylab.html` inside `html/` on the card.

**If X never starts:** wait the full two minutes. Watch the text console for `Installing packages from boot media cache`. If that warning appears, `cache/` did not copy onto the card.

---

## 12. Power off

Normal power-off is enough.

Chromium profile, bash history, and `/tmp` live in RAM. The boot partition is remounted read-only after the HTML copy. There is no `lbu commit` on the Pi.

Reuse the same card later: it still only has the OS files plus whatever is in `html/`. Swap the HTML on a Mac if you want a different tool.

---

## Notes

**Rebuild from scratch:** delete `boot/`, `cache/`, `ovl_root/` in the build folder and start from step 3.

**Why Openbox, not XFCE:** you asked for minimal. Openbox + tint2 + pcmanfm + xterm is a real desktop (menu, panel, files, terminal) without XFCE’s extra stack.

**Why packages install on every boot:** that **is** Alpine diskless. The install source is the SD card, not the network. This is the same cache pattern the previous guide used, which is the standard Alpine RAM-only model.

**Why we did not strip the kernel modloop:** firmware overlays (`disable-wifi` / `disable-bt`) plus `ip=off` plus no networking service is the proven, standard lock-down. Rebuilding modloop is fragile across Alpine versions. We can add it later if you want a second layer.

**Intel Macs** are not covered. This workflow needs native `linux/arm64` containers on Apple Silicon.
