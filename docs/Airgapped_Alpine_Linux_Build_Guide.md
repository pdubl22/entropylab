# Airgapped Alpine Linux Build Guide

**RAM-only Alpine for Raspberry Pi 4 / 5 — built entirely on macOS**

Phased build: get a **stock login** first, then a **minimal overlay** (install packages from `cache/` into RAM), then optionally desktop + Chromium. This matches what actually works on real hardware.

Optimized for **Apple Silicon Macs** (M1–M4) running **macOS Tahoe**, using **OrbStack** (or Docker Desktop).

> **Hardware:** Raspberry Pi 4 or 5 with **4 GB RAM or more** (8 GB comfortable). The whole OS plus Chromium lives in RAM.

---

## What this build is (and is not)

| Item | Choice |
| --- | --- |
| Alpine | `latest-stable`, aarch64 |
| Mode | Diskless — root filesystem in tmpfs (RAM) |
| Phase 1 | Stock Alpine login (`root`, empty password) |
| Phase 2 | Minimal overlay: install `cache/*.apk` into RAM |
| Phase 3 (optional) | Openbox desktop + Chromium + `file://` HTML |
| Network on the Pi | Not required; Wi-Fi/BT can be disabled in firmware later |
| Persistence | None. Normal power-off is enough |

**Left out on purpose (for now):** autologin, early udev, custom `inittab`, kiosk mode, local web server.

### How diskless boot works

1. Pi firmware reads the FAT32 card (`config.txt` points at `boot/vmlinuz-rpi` and `boot/initramfs-rpi`).
2. Initramfs creates a tmpfs root and loads `boot/modloop-rpi`.
3. If present at the **card root**, `*.apkovl.tar.gz` is applied.
4. OpenRC runs; a `local.d` script can `apk add` from `cache/` on the card (no network).
5. You get a login prompt (Phase 1/2) or a desktop (Phase 3).

Package install on every boot **is** normal for Alpine diskless. Source is the SD card, not the network.

---

## Critical rules (learned the hard way)

1. **Never rename an overlay to `something.apkovl.tar.gz.off`.**  
   Alpine still loads `*.apkovl.tar.gz*` and treats `off` as an encryption cipher →  
   `Cipher off is not supported` → initramfs emergency shell.  
   To disable an overlay: move it into a folder named `disabled/` on the card, or rename to a name **without** `apkovl.tar.gz` (e.g. `overlay-backup.tar.gz`).

2. **Do not replace `inittab` with `agetty` until that package is installed.**  
   Base image only has busybox `getty`. Missing `agetty` loops forever.

3. **Do not enable `udev` in sysinit before packages are installed.**  
   Prefer Alpine’s default **mdev** until `cache/` has been applied.

4. **Kernel files live under `boot/` on the card**, not the card root:
   - `boot/vmlinuz-rpi`
   - `boot/initramfs-rpi`
   - `boot/modloop-rpi`  
   `config.txt` contains `kernel=boot/vmlinuz-rpi` and `initramfs boot/initramfs-rpi`.

5. **Paste small blocks.** Giant heredocs in zsh on macOS often break. Avoid bare `#` comment lines outside heredocs (some zsh configs treat `#` as a command).

6. **HDMI:** if the screen goes black after a brief cursor, try `hdmi_safe=1` and **no** `vc4-kms-v3d` until text console works. Add KMS later for X.

7. **USB:** host can work while a hub keyboard (e.g. Apple keyboard with extra ports) still misbehaves. Prefer a simple wired keyboard on a USB2 port. Boot with no USB devices if the kernel seems stuck early.

---

## Requirements

**Mac:** Apple Silicon, OrbStack/Docker, ~3 GB free, SD card you can erase.  
**Pi:** 4 or 5, 4 GB+ RAM, HDMI, keyboard, **no Ethernet required**.  
**You supply last:** `entropylab.html` into `html/` on the card.

---

## Phase 0 — Mac folders and packages

```zsh
mkdir -p boot cache html disabled
```

Fetch packages (Mac uses network; Pi will not):

```zsh
open -a OrbStack 2>/dev/null || true
until docker info >/dev/null 2>&1; do sleep 1; done

docker run --rm -v "$(pwd):/work" -w /work --platform linux/arm64 alpine:latest sh -c '
  echo "https://dl-cdn.alpinelinux.org/alpine/latest-stable/main" > /etc/apk/repositories
  echo "https://dl-cdn.alpinelinux.org/alpine/latest-stable/community" >> /etc/apk/repositories
  apk update
  apk fetch --recursive -o /work/cache \
    eudev eudev-openrc \
    dbus dbus-openrc \
    shadow \
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
ls cache | wc -l
```

Expect a large number (often 200+).

Download Alpine RPi boot files:

```zsh
RPI_FILE=$(docker run --rm --platform linux/arm64 alpine:latest sh -c '
  wget -qO- https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/aarch64/latest-releases.yaml \
    | awk "\$0 ~ /flavor: alpine-rpi/ {inr=1} inr && \$1 == \"file:\" && \$2 ~ /\\.tar\\.gz$/ {print \$2; exit}"
')
echo "Downloading $RPI_FILE ..."
curl -fL --progress-bar -o rpi.tar.gz \
  "https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/aarch64/$RPI_FILE"
tar -xzf rpi.tar.gz -C boot/
rm rpi.tar.gz
ls boot/boot/vmlinuz-rpi boot/boot/initramfs-rpi boot/boot/modloop-rpi 2>/dev/null || ls boot/vmlinuz-rpi boot/initramfs-rpi boot/modloop-rpi 2>/dev/null || ls -la boot | head
```

(Alpine’s tarball layout may place `vmlinuz-rpi` under `boot/boot/` or `boot/` depending on extract path. After extract, the **card root** must end up with `config.txt` + a `boot/` dir containing the kernel files.)

Safe HDMI for first boots (write into the extracted tree that becomes the card root):

```zsh
# If config.txt is at boot/config.txt after extract:
CFG=boot/config.txt
[ -f "$CFG" ] || CFG=boot/boot/config.txt
# usercfg next to config.txt
USERCFG=$(dirname "$CFG")/usercfg.txt
cat > "$USERCFG" << 'EOF'
hdmi_force_hotplug=1
hdmi_safe=1
disable_overscan=1
EOF
```

Optional later (after text console works): add KMS and radio disables to `usercfg.txt`.

---

## Phase 1 — Stock card (no overlay)

Copy extracted Alpine files + `cache/` to a FAT32 MBR card labeled e.g. `ENTROPYLAB`.  
**Do not** put any `*.apkovl.tar.gz` on the card root yet.

Boot the Pi. You should get:

```text
login:
```

Login: `root` / empty password → `localhost:~#`

Check:

```sh
uname -a
free -m
mount | grep mmc
ls /media
ls /media/*/cache 2>/dev/null | wc -l
```

Boot partition may appear as `/media/mmcblk0p1` or `/media/mmcblk01` (or similar). Cache count should match what you put on the card (~200+).

If you see `Cipher ... is not supported` or an initramfs emergency shell, an apkovl filename is wrong — remove every `*apkovl*` from the **card root**.

---

## Phase 2 — Minimal overlay (packages only)

**Only:** hostname + enable `local` + install `cache/*.apk`.  
**No** custom `inittab`, **no** udev in sysinit, **no** autologin, **no** X.

### On the Mac

```zsh
rm -rf min_ovl
mkdir -p min_ovl/etc/local.d min_ovl/etc/runlevels/default

echo "entropylab" > min_ovl/etc/hostname

cat > min_ovl/etc/local.d/00-install-cache.start << 'EOF'
#!/bin/sh
echo "Installing packages from boot media cache into RAM..."
CACHE_FOUND=0
for MOUNT_POINT in /media/mmcblk0p1 /media/mmcblk01 /media/ENTROPYLAB /media/ALPINE /media/*; do
  [ -d "$MOUNT_POINT/cache" ] || continue
  if ls "$MOUNT_POINT/cache"/*.apk >/dev/null 2>&1; then
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
chmod +x min_ovl/etc/local.d/00-install-cache.start

ln -sf /etc/init.d/local min_ovl/etc/runlevels/default/local

tar -czf entropylab.apkovl.tar.gz -C min_ovl .
ls -l entropylab.apkovl.tar.gz
```

Copy **only** this file to the **card root** (not into `disabled/`):

```zsh
VOL=/Volumes/ENTROPYLAB
cp entropylab.apkovl.tar.gz "$VOL/"
mkdir -p "$VOL/disabled"
# keep old experiments out of the root
mv "$VOL"/*.apkovl.tar.gz.off "$VOL/disabled/" 2>/dev/null || true
ls -la "$VOL"/*apkovl* 2>/dev/null
sync
diskutil eject "$VOL"
```

### On the Pi

Boot, wait through package install (can take 1–2 minutes), login `root`.

```sh
hostname
command -v startx
command -v chromium-browser
command -v openbox
apk info -e chromium openbox xorg-server 2>/dev/null
```

If those commands exist, Phase 2 succeeded.

---

## Phase 3 — Desktop (only after Phase 2 works)

Add a **second** overlay pass or extend `min_ovl` carefully:

- User `entropylab` created in `local.d` **after** packages install
- Xorg config, Openbox, Chromium flags
- HTML copy from `html/` on the card into `/tmp/html`
- Autologin only with **busybox getty** + `/usr/local/bin/entropylab-autologin` (`login -f entropylab`), never `agetty` until you confirm `/sbin/agetty` exists
- Enable KMS in `usercfg.txt` only after text console is reliable:

```text
dtoverlay=vc4-kms-v3d
max_framebuffers=2
hdmi_force_hotplug=1
```

Keep `dtoverlay=disable-wifi` / `disable-bt` and `ip=off` as optional hardening once the desktop is stable.

---

## SD card layout (final)

Card root should look like:

- `config.txt`, `cmdline.txt`, `usercfg.txt`
- `boot/` → `vmlinuz-rpi`, `initramfs-rpi`, `modloop-rpi`
- `cache/` → many `.apk` files
- `html/` → `entropylab.html` (Phase 3)
- `entropylab.apkovl.tar.gz` (one file, correct name)
- `disabled/` → old overlays only
- firmware / `*.dtb` / `overlays/` from Alpine tarball

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `Cipher off is not supported` | Remove `*.apkovl.tar.gz.*` from card root; use `disabled/` |
| Black screen after cursor | `hdmi_safe=1`, remove KMS temporarily |
| `can't run agetty` | Use busybox `getty`; do not customize inittab early |
| `can't create /dev/null` | Do not force udev before packages; use stock mdev |
| USB messages but no login | Unplug USB devices; fix apkovl name; wait for stock login |
| Keyboard LED dead but USB logs | Try non-hub wired keyboard on USB2; Apple hub keyboards are flaky |
| `cache` not found on Pi | `mount \| grep mmc`; look under `/media/*` |

---

## Notes

**Why phased:** A full desktop overlay in one step caused black screens, missing agetty, broken `/dev`, and the `.off` cipher trap. Stock → minimal packages → desktop is slower to write and much faster to debug.

**Why packages every boot:** Alpine diskless design. Install source is the SD `cache/`, not the network.

**Intel Macs:** not covered (need `linux/arm64` containers).
