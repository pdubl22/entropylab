# 🛡️ Hardened macOS Kiosk User Guide

**Secure, Low-Persistence Runtime Environment for Apple Silicon**

This guide outlines the process for creating a dedicated, hardened macOS user account designed for a specific application (EntropyLab). The focus is on **minimizing the software attack surface** and **preventing data persistence** by forcing the browser to operate out of a volatile RAM directory.

## 🎯 Project Goal

To create a "disposable" user session where:
1. **No Onboarding:** The user is dropped directly into the environment without iCloud/Siri prompts.
2. **Zero Browser Persistence:** Chromium data is stored in `/tmp` and wiped upon reboot.
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
These commands trick macOS into thinking the "Welcome" process is complete and disable invasive features that could lead to data leaks or software prompts.

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
# Disable Spotlight suggestions (prevents searching for system tools)
sudo -u $USER_NAME defaults write com.apple.Spotlight ShowAllSuggestions -bool false
# Remove recently used apps from Dock
sudo -u $USER_NAME defaults write com.apple.dock show-recents -bool false
# Disable 'Get Info' to prevent inspection of folder permissions
sudo -u $USER_NAME defaults write com.apple.finder DisableGetInfo -bool true
# Hide desktop icons for a cleaner, appliance-like feel
sudo -u $USER_NAME defaults write com.apple.finder CreateDesktop -bool false

echo "✅ System flags applied."
```

### 3. The Hardened Browser Wrapper
To ensure the browser is truly "RAM-only," we create a launch script that redirects the user profile to the `/tmp` directory. This ensures that history, cookies, and cache are deleted when the machine is restarted.

```zsh
# Create a local bin directory for the user
sudo mkdir -p /Users/entropylab/bin
sudo chown entropylab:staff /Users/entropylab/bin

# Create the launch script
sudo tee /Users/entropylab/bin/launch_browser.sh << 'EOF'
#!/bin/zsh
# Clear existing chrome tmp data to ensure a fresh session
rm -rf /tmp/chrome

# Launch Chromium/Chrome with hardening flags
# --user-data-dir=/tmp/chrome is the key to zero persistence
open -a "Google Chrome" --args \
    --incognito \
    --no-first-run \
    --disable-sync \
    --disable-extensions \
    --disable-component-update \
    --disable-notifications \
    --user-data-dir=/tmp/chrome \
    "http://127.0.0.1:8080/entropylab.html"
EOF

# Set permissions
sudo chmod +x /Users/entropylab/bin/launch_browser.sh
sudo chown -R entropylab:staff /Users/entropylab/bin
```

### 4. Automation: Launch on Login
We create a `launchd` agent so the browser starts automatically the moment the `entropylab` user logs in.

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
The user will have full access to USB drives. However, the first time the browser attempts to save a file to or read from a USB drive, macOS will show a **TCC (Transparency, Consent, and Control)** popup. 
- **Action:** Log in as `entropylab` once, perform a USB action, and click **"Allow."** This permission is permanent for that user.

### 🧹 Data Persistence Summary
| Component | Storage Location | Persistence |
| :--- | :--- | :--- |
| **System Files** | NVMe (Read-Only System Vol) | Permanent |
| **User Settings** | `/Users/entropylab/Library` | Permanent |
| **Browser Profile** | `/tmp/chrome` | **Volatile (Wiped on Reboot)** |
| **Browser History** | Incognito Mode | **Volatile (Wiped on Close)** |

### 🛠️ Maintenance
To completely reset the user environment, you can simply delete the user and run the script again:
```zsh
sudo sysadminctl -deleteUser entropylab
```
