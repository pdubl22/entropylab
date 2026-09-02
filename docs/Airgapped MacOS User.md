# 🛡️ Hardened macOS Kiosk User Guide

**Secure, Low-Persistence Runtime Environment for Apple Silicon**

This guide outlines the process for creating a dedicated, hardened macOS user account designed for a specific application (EntropyLab). The focus is on **minimizing the software attack surface** and **preventing data persistence** by forcing the browser to operate out of a volatile RAM directory.

## 🎯 Project Goal

To create a "disposable" user session where:
1. **No Onboarding:** The user is dropped directly into the environment without iCloud/Siri prompts.
2. **Zero Browser Persistence:** Chromium data is stored in a true RAM disk and wiped upon reboot.
3. **Reduced Footprint:** System features (Spotlight, Dock recents) are disabled to minimize background noise.
4. **Privilege Limitation:** The user is a "Standard User," preventing the installation of system-wide software or modification of system files.

---

## 🛠️ Deployment Workflow

Execute these steps from an **Administrator** account.

### 1. User Creation
We use `sysadminctl` to ensure the user is created with the correct Secure Token for Apple Silicon hardware.

```zsh
# Replace 'UserPassword123' with your desired password
sudo sysadminctl -addUser entropylab -fullName "EntropyLab" -password "UserPassword123"
```

### 2. System Silence & Hardening
These commands trick macOS into thinking the "Welcome" process is complete and disable invasive features.

```zsh
USER_NAME="entropylab"

echo "Applying hardening flags for $USER_NAME..."

# --- Skip Onboarding & Popups ---
sudo -u $USER_NAME defaults write com.apple.setupassistant SetupDone -bool true
sudo -u $USER_NAME defaults write com.apple.Siri SiriEnabled -bool false
sudo -u $USER_NAME defaults write com.apple.icloud iCloudDriveEnabled -bool false
sudo -u $USER_NAME defaults write com.apple.ApplePay SetupCompleted -bool true
sudo -u $USER_NAME defaults write com.apple.screentime SetupCompleted -bool true

# --- UI & Privacy Hardening ---
sudo -u $USER_NAME defaults write com.apple.Spotlight ShowAllSuggestions -bool false
sudo -u $USER_NAME defaults write com.apple.dock show-recents -bool false
sudo -u $USER_NAME defaults write com.apple.finder DisableGetInfo -bool true
sudo -u $USER_NAME defaults write com.apple.finder CreateDesktop -bool false

echo "✅ System flags applied."
```

### 3. The RAM Disk Engine (System Level)
Because only the root user can create a RAM disk, we create a **LaunchDaemon**. This ensures the RAM disk is ready and waiting before the user even logs in.

```zsh
# Create the script that creates the RAM disk
sudo tee /usr/local/bin/create_ramdisk.sh << 'EOF'
#!/bin/zsh
# Create a 256MB RAM Disk
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
Now we create the launch script for the user. Since the RAM disk is already created by the system, the user only needs to launch the browser and point it to that volume.

```zsh
# Create a local bin directory for the user
sudo mkdir -p /Users/entropylab/bin
sudo chown entropylab:staff /Users/entropylab/bin

# Create the launch script
sudo tee /Users/entropylab/bin/launch_browser.sh << 'EOF'
#!/bin/zsh

# Launch Chromium/Chrome using the system-created RAM disk
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

# Set permissions
sudo chmod +x /Users/entropylab/bin/launch_browser.sh
sudo chown -R entropylab:staff /Users/entropylab/bin
```

### 5. Automation: Launch on Login
We create a `launchd` agent to trigger the browser script upon login.

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

---

## 📝 Final Operational Notes

### 🔑 USB Access
The user will have full access to USB drives. However, the first time the browser attempts to access a USB drive, macOS will show a **TCC (Transparency, Consent, and Control)** popup. 
- **Action:** Log in as `entropylab` once, perform a USB action, and click **"Allow."** This permission is permanent for that user.

### 🧹 Data Persistence Summary
| Component | Storage Location | Persistence |
| :--- | :--- | :--- |
| **System Files** | NVMe (Read-Only System Vol) | Permanent |
| **User Settings** | `/Users/entropylab/Library` | Permanent |
| **Browser Profile** | `/Volumes/RAMDISK/chrome` | **Volatile (Wiped on Reboot)** |
| **Browser History** | Incognito Mode | **Volatile (Wiped on Close)** |

### 🛠️ Maintenance
To completely reset the user environment, you can delete the user:
```zsh
sudo sysadminctl -deleteUser entropylab
```
To remove the system-level RAM disk engine:
```zsh
sudo rm /Library/LaunchDaemons/com.entropylab.ramdisk.plist
sudo rm /usr/local/bin/create_ramdisk.sh
```
