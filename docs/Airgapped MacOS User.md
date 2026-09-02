# 🛡️ Hardened macOS Kiosk User Guide

**Secure, Low-Persistence Runtime Environment for Apple Silicon**

This guide outlines the process for creating a dedicated, hardened macOS user account designed for the **EntropyLab** application. The focus is on **minimizing the software attack surface** and **preventing data persistence** by forcing the browser to operate out of a volatile RAM directory.

## 🎯 Project Goal

To create a "disposable" user session where:
1. **No Onboarding:** The user is dropped directly into the environment without iCloud/Siri prompts.
2. **Zero Browser Persistence:** Chromium data is stored in a true RAM disk and wiped upon reboot.
3. **Reduced Footprint:** System features (Spotlight, Dock recents, Desktop icons) are disabled.
4. **Privilege Limitation:** The user is a "Standard User," preventing system-wide modifications.
5. **Easy Exit:** A "Self-Destruct" button allows the user to wipe the environment and reset system power settings instantly.

---

## 🛠️ Deployment Workflow

**Execute all following steps from an Administrator account.**

### 1. User Creation
We use `sysadminctl` to ensure the user is created with the correct Secure Token for Apple Silicon hardware.

```zsh
# Replace 'UserPassword123' with your desired password
sudo sysadminctl -addUser entropylab -fullName "EntropyLab" -password "UserPassword123"
```

### 2. System Silence & Hardening
These commands trick macOS into thinking the "Welcome" process is complete and disable features that could lead to data leaks or distractions.

```zsh
USER_NAME="entropylab"

echo "Applying hardening flags for $USER_NAME..."

# --- Skip Onboarding & Popups ---
sudo -u $USER_NAME defaults write com.apple.setupassistant SetupDone -bool true
sudo -u $USER_NAME defaults write com.apple.Siri SiriEnabled -bool false
sudo -u $USER_NAME defaults write com.apple.icloud iCloudDriveEnabled -bool false
sudo -u $USER_NAME defaults write com.apple.ApplePay SetupCompleted -bool true
sudo -u $USER_NAME defaults write com.apple.screentime SetupCompleted -bool true

# --- System-Wide Silence ---
# Disable automatic software update checks to reduce telemetry and popups
sudo softwareupdate --schedule off

# --- UI & Privacy Hardening ---
sudo -u $USER_NAME defaults write com.apple.Spotlight ShowAllSuggestions -bool false
sudo -u $USER_NAME defaults write com.apple.dock show-recents -bool false
sudo -u $USER_NAME defaults write com.apple.finder DisableGetInfo -bool true
sudo -u $USER_NAME defaults write com.apple.finder CreateDesktop -bool false
sudo -u $USER_NAME defaults write com.apple.finder ShowExternalVolumesOnDesktop -bool false
sudo -u $USER_NAME defaults write com.apple.finder ShowHardDisksOnDesktop -bool false

# --- Power Management ---
# Disable sleep, screen saver, and hibernation to prevent interruptions 
# and prevent RAM contents from being written to the NVMe sleepimage.
sudo pmset -a sleep 0 disablesleep 1 hibernatemode 0
sudo -u $USER_NAME defaults write com.apple.screensaver idleTime -int 0

echo "✅ System flags applied."
```

### 3. The RAM Disk Engine (System Level)
Because only the root user can create a RAM disk, we create a **LaunchDaemon**. This ensures the RAM disk is ready and waiting before the user even logs in.

```zsh
# Create the script that creates the RAM disk
sudo tee /usr/local/bin/create_ramdisk.sh << 'EOF'
#!/bin/zsh
# Create a 256MB RAM Disk (524288 blocks)
RAM_DISK_SIZE=524288 
diskutil create "disk${RAM_DISK_SIZE}" 0 HFS+ "RAMDISK" $RAM_DISK_SIZE
mkdir -p /Volumes/RAMDISK/chrome
chmod 777 /Volumes/RAMDISK/chrome
EOF

sudo chmod +x /usr/local/bin/create_ramdisk.sh

# Create the LaunchDaemon to run this at boot
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
</dict>
</plist>
EOF

sudo chown root:wheel /Library/LaunchDaemons/com.entropylab.ramdisk.plist
```

### 4. The Hardened Browser Wrapper (User Level)
This script launches the browser and forces it to use the RAM disk for all profile data.

```zsh
# Create a local bin directory for the user
sudo mkdir -p /Users/entropylab/bin
sudo chown entropylab:staff /Users/entropylab/bin

# Create the launch script
sudo tee /Users/entropylab/bin/launch_browser.sh << 'EOF'
#!/bin/zsh
open -a "Google Chrome" --args \
    --incognito \
    --no-first-run \
    --disable-sync \
    --disable-extensions \
    --disable-component-update \
    --disable-notifications \
    --disable-software-rasterizer \
    --disable-dev-shm-usage \
    --user-data-dir=/Volumes/RAMDISK/chrome \
    "http://127.0.0.1:8080/entropylab.html"
EOF

sudo chmod +x /Users/entropylab/bin/launch_browser.sh
sudo chown -R entropylab:staff /Users/entropylab/bin
```

### 5. Automation: Launch on Login
This creates a `launchd` agent that triggers the browser script immediately upon login.

```zsh
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

sudo chown entropylab:staff /Users/entropylab/Library/LaunchAgents/com.entropylab.browser.plist
```

### 6. The "Self-Destruct" Mechanism
To allow the restricted user to wipe the environment and reset the system without needing an admin password, we implement a privileged cleanup script.

```zsh
# 1. Create the root-level cleanup script
sudo tee /usr/local/bin/cleanup_entropylab.sh << 'EOF'
#!/bin/zsh
echo "🧹 Self-Destruct Initiated..."

# Reset System Power Settings (Return to 10m sleep)
sudo pmset -a sleep 10 disablesleep 0 hibernatemode 3

# Remove the System-Level RAM Disk Engine
sudo rm -f /Library/LaunchDaemons/com.entropylab.ramdisk.plist
sudo rm -f /usr/local/bin/create_ramdisk.sh

# Unmount the RAM Disk
if mount | grep -q "RAMDISK"; then
    diskutil eject /Volumes/RAMDISK
fi

# Delete the entropylab user (Force logout and wipe)
sudo sysadminctl -deleteUser entropylab
EOF

sudo chmod +x /usr/local/bin/cleanup_entropylab.sh

# 2. Grant the entropylab user passwordless sudo rights ONLY for this script
echo "entropylab ALL=(ALL) NOPASSWD: /usr/local/bin/cleanup_entropylab.sh" | sudo tee /etc/sudoers.d/entropylab_cleanup

# 3. Create the double-clickable .command shortcut on the user's desktop
sudo tee /Users/entropylab/Desktop/Self-Destruct.command << 'EOF'
#!/bin/zsh
sudo /usr/local/bin/cleanup_entropylab.sh
EOF

sudo chmod +x /Users/entropylab/Desktop/Self-Destruct.command
sudo chown entropylab:staff /Users/entropylab/Desktop/Self-Destruct.command
```

---

## 📝 Final Operational Notes

### 🔑 USB Access
The user has full access to USB drives. However, the first time the browser attempts to access a USB drive, macOS will show a **TCC (Transparency, Consent, and Control)** popup. 
- **Action:** Log in as `entropylab` once, perform a USB action, and click **"Allow."** This permission is permanent for that user.

### 🧹 Data Persistence Summary
| Component | Storage Location | Persistence |
| :--- | :--- | :--- |
| **System Files** | NVMe (Read-Only System Vol) | Permanent |
| **User Settings** | `/Users/entropylab/Library` | Permanent |
| **Browser Profile** | `/Volumes/RAMDISK/chrome` | **Volatile (Wiped on Reboot)** |
| **Browser History** | Incognito Mode | **Volatile (Wiped on Close)** |

### 🛠️ Manual Maintenance
If the self-destruct button is not used, you can manually remove the environment via:
```zsh
sudo sysadminctl -deleteUser entropylab
sudo rm /Library/LaunchDaemons/com.entropylab.ramdisk.plist
sudo rm /usr/local/bin/create_ramdisk.sh
sudo rm /etc/sudoers.d/entropylab_cleanup
sudo pmset -a sleep 10 disablesleep 0 hibernatemode 3
```
