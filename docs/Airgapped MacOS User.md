# Airgapped macOS user for EntropyLab

Dedicated **standard** user on Apple Silicon. Chromium/Chrome profile lives on a RAM disk. The app is the self-contained [`entropylab.html`](https://github.com/OogaBoogaX/entropylab/blob/rock/entropylab.html) file (logos/JS/WASM already inlined). Opened as `file://` — no local web server.

This is a **smaller** attack surface than a normal Mac login. It is **not** as small as the Alpine Pi image. macOS still has a huge TCB. Use a machine that stays offline after setup.

The browser is **fullscreen, not kiosk**, so Save As to USB works.

> [!CAUTION]
> Run every command from an **admin** account that already has a Secure Token (the account you created the Mac with).
>
> Install **Chromium or Google Chrome** from a verified .dmg **before** you pull the network. This guide does not download a browser.
>
> `pmset` and the RAM-disk LaunchDaemon are **machine-wide**. Use Self-Destruct (or the manual cleanup) when you want the Mac back to normal sleep.
>
> Run the script with **zsh**, not bash: `zsh "Airgapped MacOS User.sh"`

## Automated script

A generated shell script is kept in sync with this guide:

- Source of truth: this Markdown file
- Generated file: [`Airgapped MacOS User.sh`](./Airgapped%20MacOS%20User.sh)
- GitHub Action extracts every fenced block tagged `zsh extract` (sections 1–6) and writes the `.sh`
- You can still copy any code block from this page into a terminal; the `extract` tag is only a marker for the Action and does not appear in the copied commands

Run the generated script from an admin account that already has a Secure Token.

## Goal

1. Standard user, no iCloud/Siri first-run circus.
2. Browser profile on a RAM disk, gone at reboot.
3. `file://` EntropyLab HTML, no listening HTTP port.
4. USB save works (native file picker).
5. One Dock item runs a root-owned cleanup script via a **narrow** sudoers rule.

---

## 1. Copy the app and create the user

```zsh extract
# Fresh Apple Silicon installs often lack /usr/local/bin
sudo mkdir -p /usr/local/bin

# Self-contained artifact. Prefer a release + SHA256SUMS.txt for real funds.
curl -fL -o /tmp/entropylab.html \
  https://raw.githubusercontent.com/OogaBoogaX/entropylab/rock/entropylab.html

# Prompt for the kiosk password instead of baking one into history.
# Nothing will echo while you type — that is normal.
echo "Password for the new standard user 'entropylab':"
read -s EL_PASS
echo

if [ -z "${EL_PASS}" ]; then
  echo "error: empty password — re-run and type a password, then press Enter" >&2
  exit 1
fi

# Remove a half-created user from a previous failed run (ignore errors)
sudo sysadminctl -deleteUser entropylab -secure 2>/dev/null || true
sudo rm -rf /Users/entropylab 2>/dev/null || true

sudo sysadminctl -addUser entropylab \
  -fullName "EntropyLab" \
  -password "$EL_PASS" \
  -home /Users/entropylab \
  -shell /bin/zsh

# sysadminctl may print an FDE warning even when -password was set; that can be ignored
# if the next lines succeed. Ensure the home directory really exists.
if [ ! -d /Users/entropylab ]; then
  sudo createhomedir -c -u entropylab
fi
if [ ! -d /Users/entropylab ]; then
  echo "error: /Users/entropylab was not created" >&2
  exit 1
fi

unset EL_PASS

sudo mkdir -p /Users/entropylab/bin /Users/entropylab/Library/LaunchAgents
sudo cp /tmp/entropylab.html /Users/entropylab/entropylab.html
sudo chown -R entropylab:staff /Users/entropylab
sudo chmod 644 /Users/entropylab/entropylab.html
# Optional: lock the HTML so the standard user cannot overwrite it
sudo chflags uchg /Users/entropylab/entropylab.html
```

`sysadminctl -addUser` (run by a Secure Token admin, with `-password`) is the supported way to bootstrap a token for the new user on Apple Silicon. Do not use `dscl` for this.

If `sysadminctl` prints `No clear text password or interactive option was specified`, check that you actually typed a password at the prompt (it does not echo). The script now aborts on an empty password. A successful run still creates `/Users/entropylab` (via `createhomedir` if needed).

## 2. Skip first-run and quiet the UI

Do **not** hide `/Volumes` from the file picker. `CreateDesktop` only hides desktop *icons*; Chrome's Save As still sees USB disks under `/Volumes`.

```zsh extract
U=entropylab

sudo -u "$U" defaults write com.apple.SetupAssistant DidSeeCloudSetup -bool true
sudo -u "$U" defaults write com.apple.SetupAssistant SkipCloudSetup -bool true
sudo -u "$U" defaults write com.apple.SetupAssistant DidSeeSiriSetup -bool true
sudo -u "$U" defaults write com.apple.SetupAssistant DidSeePrivacy -bool true
sudo -u "$U" defaults write com.apple.SetupAssistant DidSeeAppearanceSetup -bool true
sudo -u "$U" defaults write com.apple.SetupAssistant DidSeeScreenTime -bool true
sudo -u "$U" defaults write com.apple.SetupAssistant DidSeeAccessibility -bool true

sudo -u "$U" defaults write com.apple.Siri StatusMenuVisible -bool false
sudo -u "$U" defaults write com.apple.Siri SiriEnabled -bool false
sudo -u "$U" defaults write com.apple.dock show-recents -bool false
sudo -u "$U" defaults write com.apple.finder CreateDesktop -bool false

# Sleep image would write RAM to NVMe. Machine-wide; Self-Destruct restores this.
sudo pmset -a sleep 0 disablesleep 1 hibernatemode 0
sudo -u "$U" defaults write com.apple.screensaver idleTime -int 0
```

Leave Wi-Fi/Bluetooth off from the menu bar (or `networksetup -setairportpower en0 off`) **on the machine**, not only in this user. This guide does not toggle radios automatically so we cannot brick a Mac that still needs admin network.

## 3. RAM disk (LaunchDaemon, root, idempotent)

256 MB is too small for a Chrome profile. Default is **1 GiB**. `ram://` units are 512-byte sectors (`MB * 2048`).

```zsh extract
sudo mkdir -p /usr/local/bin

sudo tee /usr/local/bin/create_ramdisk.sh << 'EOF'
#!/bin/zsh
set -euo pipefail

OWNER=entropylab
VOL=/Volumes/RAMDISK
SECTORS=$((1024 * 2048))  # 1 GiB

if [ -d "$VOL/chrome" ]; then
  chown -R "$OWNER":staff "$VOL"
  chmod 700 "$VOL/chrome"
  exit 0
fi

DEV=$(hdiutil attach -nomount "ram://${SECTORS}" | tr -d '[:space:]')
if ! diskutil erasevolume APFS RAMDISK "$DEV"; then
  diskutil erasevolume HFS+ RAMDISK "$DEV"
fi

mkdir -p "$VOL/chrome"
chown -R "$OWNER":staff "$VOL"
chmod 700 "$VOL/chrome"
EOF
sudo chmod 755 /usr/local/bin/create_ramdisk.sh
sudo chown root:wheel /usr/local/bin/create_ramdisk.sh

sudo tee /Library/LaunchDaemons/com.entropylab.ramdisk.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.entropylab.ramdisk</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/create_ramdisk.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/var/log/entropylab-ramdisk.log</string>
    <key>StandardErrorPath</key>
    <string>/var/log/entropylab-ramdisk.log</string>
</dict>
</plist>
EOF
sudo chown root:wheel /Library/LaunchDaemons/com.entropylab.ramdisk.plist
sudo chmod 644 /Library/LaunchDaemons/com.entropylab.ramdisk.plist

sudo launchctl bootout system/com.entropylab.ramdisk 2>/dev/null || true
sudo launchctl bootstrap system /Library/LaunchDaemons/com.entropylab.ramdisk.plist
sudo launchctl enable system/com.entropylab.ramdisk
sudo launchctl kickstart -k system/com.entropylab.ramdisk
```

Confirm `/Volumes/RAMDISK/chrome` exists and is owned by `entropylab` before continuing.

## 4. Browser wrapper (wait for RAM disk, fullscreen, file://)

Prefers Chromium if installed, else Google Chrome. No kiosk flag. No `--disable-software-rasterizer` (hurts Mac GPU). No local HTTP server.

```zsh extract
sudo tee /Users/entropylab/bin/launch_browser.sh << 'EOF'
#!/bin/zsh
set -euo pipefail

for i in {1..30}; do
  [ -d /Volumes/RAMDISK/chrome ] && break
  sleep 1
done
if [ ! -d /Volumes/RAMDISK/chrome ]; then
  osascript -e 'display notification "RAM disk missing" with title "EntropyLab"'
  exit 1
fi

APP="/Applications/Chromium.app"
if [ ! -d "$APP" ]; then
  APP="/Applications/Google Chrome.app"
fi
if [ ! -d "$APP" ]; then
  osascript -e 'display notification "Install Chromium or Google Chrome first" with title "EntropyLab"'
  exit 1
fi

HTML="/Users/entropylab/entropylab.html"

open -a "$APP" --args \
  --incognito \
  --no-first-run \
  --no-default-browser-check \
  --disable-sync \
  --disable-extensions \
  --disable-component-update \
  --disable-notifications \
  --disable-infobars \
  --noerrdialogs \
  --disable-session-crashed-bubble \
  --disable-features=TranslateUI \
  --password-store=basic \
  --use-mock-keychain \
  --start-fullscreen \
  --user-data-dir=/Volumes/RAMDISK/chrome \
  "file://${HTML}"
EOF

sudo chown -R entropylab:staff /Users/entropylab/bin
sudo chmod 755 /Users/entropylab/bin/launch_browser.sh
```

## 5. Launch agent (runs at login)

Do not `bootstrap gui/` here -- that user is not logged in yet. The plist in `~/Library/LaunchAgents` loads on next login.

```zsh extract
sudo tee /Users/entropylab/Library/LaunchAgents/com.entropylab.browser.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.entropylab.browser</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/entropylab/bin/launch_browser.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
EOF
sudo chown -R entropylab:staff /Users/entropylab/Library
sudo chmod 644 /Users/entropylab/Library/LaunchAgents/com.entropylab.browser.plist
```

## 6. Self-Destruct (narrow sudoers)

The `.command` file is the only thing `entropylab` may run as root. The script is root-owned and not writable by that user. The script **itself** runs as root, so it must **not** call `sudo` again (the kiosk user has no general sudo).

```zsh extract
sudo mkdir -p /usr/local/bin

sudo tee /usr/local/bin/cleanup_entropylab.sh << 'EOF'
#!/bin/zsh
set -eu

pmset -a sleep 10 disablesleep 0 hibernatemode 3

launchctl bootout system/com.entropylab.ramdisk 2>/dev/null || true
rm -f /Library/LaunchDaemons/com.entropylab.ramdisk.plist
rm -f /usr/local/bin/create_ramdisk.sh

if mount | grep -q ' /Volumes/RAMDISK '; then
  diskutil eject /Volumes/RAMDISK || true
fi

# Drop the sudoers rule before deleting the user
rm -f /etc/sudoers.d/entropylab_cleanup

chflags nouchg /Users/entropylab/entropylab.html 2>/dev/null || true
sysadminctl -deleteUser entropylab -secure

/sbin/shutdown -r now
EOF
sudo chown root:wheel /usr/local/bin/cleanup_entropylab.sh
sudo chmod 755 /usr/local/bin/cleanup_entropylab.sh

printf '%s\n' 'entropylab ALL=(root) NOPASSWD: /usr/local/bin/cleanup_entropylab.sh' \
  | sudo tee /etc/sudoers.d/entropylab_cleanup
sudo chown root:wheel /etc/sudoers.d/entropylab_cleanup
sudo chmod 440 /etc/sudoers.d/entropylab_cleanup
sudo visudo -cf /etc/sudoers.d/entropylab_cleanup

sudo tee /Users/entropylab/Self-Destruct.command << 'EOF'
#!/bin/zsh
/usr/bin/sudo /usr/local/bin/cleanup_entropylab.sh
EOF
sudo chown entropylab:staff /Users/entropylab/Self-Destruct.command
sudo chmod 755 /Users/entropylab/Self-Destruct.command

sudo -u entropylab defaults write com.apple.dock persistent-others -array-add '<dict><key>tile-data</key><dict><key>file-data</key><dict><key>_CFURLString</key><string>file:///Users/entropylab/Self-Destruct.command</string><key>_CFURLStringType</key><integer>15</integer></dict></dict><key>tile-type</key><string>file-tile</string></dict>'
sudo -u entropylab killall Dock 2>/dev/null || true
```

Self-Destruct **destroys the kiosk user and the daemon**, then reboots. It is not a session logout.

---

## USB save

Chrome/Chromium uses the native macOS file picker. Removable disks show up under `/Volumes/...`.

The **first** save to a USB stick triggers a TCC prompt ("Chromium would like to access files on a removable volume"). Log in as `entropylab` once, save a test file, click Allow. That grant is per-app, per-user, and survives reboot. There is no supported command-line way to pre-grant it without a PPPC profile.

## Persistence

| What | Where | Survives reboot? |
|---|---|---|
| `entropylab.html` | `/Users/entropylab/` | Yes (optional `uchg`) |
| Browser profile / cache | `/Volumes/RAMDISK/chrome` | **No** |
| Incognito session | RAM | **No** (gone on quit) |
| TCC USB allow | user TCC db | Yes |
| Dock / SetupAssistant flags | `/Users/entropylab/Library` | Yes, until Self-Destruct |

## Manual cleanup (if you skip Self-Destruct)

```zsh
sudo launchctl bootout system/com.entropylab.ramdisk 2>/dev/null || true
sudo rm -f /Library/LaunchDaemons/com.entropylab.ramdisk.plist
sudo rm -f /usr/local/bin/create_ramdisk.sh /usr/local/bin/cleanup_entropylab.sh
sudo rm -f /etc/sudoers.d/entropylab_cleanup
sudo chflags nouchg /Users/entropylab/entropylab.html 2>/dev/null || true
sudo sysadminctl -deleteUser entropylab -secure
sudo rm -rf /Users/entropylab 2>/dev/null || true
sudo pmset -a sleep 10 disablesleep 0 hibernatemode 3
if mount | grep -q ' /Volumes/RAMDISK '; then diskutil eject /Volumes/RAMDISK; fi
```

## First login checklist

1. Fast User Switch (or log out) into `entropylab`.
2. RAM disk should already be mounted; browser should go fullscreen on EntropyLab.
3. Plug in USB → Save As → Allow TCC → confirm a file lands on the stick.
4. Reboot, confirm the Chrome profile is empty again and the HTML still opens.
