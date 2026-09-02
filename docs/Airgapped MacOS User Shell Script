#!/bin/zsh

# ==============================================================================
# 🛡️ EntropyLab macOS Provisioning Script
# This script automates the creation of a hardened user environment on MacOS.
# MUST be run with sudo: sudo zsh provision_entropylab.sh
# ==============================================================================

# Colors for output
GREEN='\033[0;32m'
BLUE='\033[0;34m'
RED='\033[0;31m'
NC='\033[0m' # No Color

# Check if script is run as root
if [[ $EUID -ne 0 ]]; then
   echo -e "${RED}❌ This script must be run as root.${NC}" 
   echo "Please use: sudo zsh provision_entropylab.sh"
   exit 1
fi

clear
echo -e "${BLUE}======================================================================"
echo -e "        🛡️  EntropyLab Hardened Environment Provisioning"
echo -e "======================================================================${NC}\n"

# --- 1. INPUTS ---
USER_NAME="entropylab"
echo -n "Enter the password for the $USER_NAME account: "
read -s USER_PASS
echo -e "\n"

if [[ -z "$USER_PASS" ]]; then
    echo -e "${RED}❌ Password cannot be empty.${NC}"
    exit 1
fi

# --- 2. USER CREATION ---
echo -e "${BLUE}[1/7] Creating User Account...${NC}"
sysadminctl -addUser $USER_NAME -fullName "EntropyLab" -password "$USER_PASS"
if [[ $? -eq 0 ]]; then
    echo -e "${GREEN}✅ User created successfully.${NC}"
else
    echo -e "${RED}❌ Failed to create user.${NC}"
    exit 1
fi

# --- 3. SYSTEM HARDENING ---
echo -e "\n${BLUE}[2/7] Applying Hardening Flags...${NC}"

# Onboarding & Popups
sudo -u $USER_NAME defaults write com.apple.setupassistant SetupDone -bool true
sudo -u $USER_NAME defaults write com.apple.Siri SiriEnabled -bool false
sudo -u $USER_NAME defaults write com.apple.icloud iCloudDriveEnabled -bool false
sudo -u $USER_NAME defaults write com.apple.ApplePay SetupCompleted -bool true
sudo -u $USER_NAME defaults write com.apple.screentime SetupCompleted -bool true

# System-Wide Silence
softwareupdate --schedule off

# UI & Privacy
sudo -u $USER_NAME defaults write com.apple.Spotlight ShowAllSuggestions -bool false
sudo -u $USER_NAME defaults write com.apple.dock show-recents -bool false
sudo -u $USER_NAME defaults write com.apple.finder DisableGetInfo -bool true
sudo -u $USER_NAME defaults write com.apple.finder CreateDesktop -bool false
sudo -u $USER_NAME defaults write com.apple.finder ShowExternalVolumesOnDesktop -bool false
sudo -u $USER_NAME defaults write com.apple.finder ShowHardDisksOnDesktop -bool false

# Power Management
pmset -a sleep 0 disablesleep 1 hibernatemode 0
sudo -u $USER_NAME defaults write com.apple.screensaver idleTime -int 0

echo -e "${GREEN}✅ System hardening applied.${NC}"

# --- 4. RAM DISK INFRASTRUCTURE ---
echo -e "\n${BLUE}[3/7] Deploying RAM Disk Engine...${NC}"

# Create RAM disk script
cat << 'EOF' > /usr/local/bin/create_ramdisk.sh
#!/bin/zsh
RAM_DISK_SIZE=524288 
RAM_DEV=$(hdiutil attach -nomount ram://$RAM_DISK_SIZE)
diskutil erasevolume HFS+ "RAMDISK" $RAM_DEV
mkdir -p /Volumes/RAMDISK/chrome
chmod 777 /Volumes/RAMDISK/chrome
EOF
chmod +x /usr/local/bin/create_ramdisk.sh

# Create LaunchDaemon
cat << 'EOF' > /Library/LaunchDaemons/com.entropylab.ramdisk.plist
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
chown root:wheel /Library/LaunchDaemons/com.entropylab.ramdisk.plist

echo -e "${GREEN}✅ RAM Disk engine deployed.${NC}"

# --- 5. BROWSER WRAPPER ---
echo -e "\n${BLUE}[4/7] Configuring Browser Wrapper...${NC}"

mkdir -p /Users/$USER_NAME/bin
cat << 'EOF' > /Users/$USER_NAME/bin/launch_browser.sh
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
chmod +x /Users/$USER_NAME/bin/launch_browser.sh
chown -R $USER_NAME:staff /Users/$USER_NAME/bin

echo -e "${GREEN}✅ Browser wrapper configured.${NC}"

# --- 6. LOGIN AUTOMATION ---
echo -e "\n${BLUE}[5/7] Setting up Login Automation...${NC}"

mkdir -p /Users/$USER_NAME/Library/LaunchAgents
cat << 'EOF' > /Users/$USER_NAME/Library/LaunchAgents/com.entropylab.browser.plist
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
chown -R $USER_NAME:staff /Users/$USER_NAME/Library/LaunchAgents

echo -e "${GREEN}✅ Login automation active.${NC}"

# --- 7. SELF-DESTRUCT MECHANISM ---
echo -e "\n${BLUE}[6/7] Installing Self-Destruct Button...${NC}"

# Root-level cleanup script
cat << 'EOF' > /usr/local/bin/cleanup_entropylab.sh
#!/bin/zsh
echo "🧹 Self-Destruct Initiated..."
sudo pmset -a sleep 10 disablesleep 0 hibernatemode 3
sudo rm -f /Library/LaunchDaemons/com.entropylab.ramdisk.plist
sudo rm -f /usr/local/bin/create_ramdisk.sh
if mount | grep -q "RAMDISK"; then
    diskutil eject /Volumes/RAMDISK
fi
sudo sysadminctl -deleteUser entropylab
sudo shutdown -r now
EOF
chmod +x /usr/local/bin/cleanup_entropylab.sh

# Sudoers exception
echo "entropylab ALL=(ALL) NOPASSWD: /usr/local/bin/cleanup_entropylab.sh" > /etc/sudoers.d/entropylab_cleanup

# Shortcut file
cat << 'EOF' > /Users/$USER_NAME/Self-Destruct.command
#!/bin/zsh
sudo /usr/local/bin/cleanup_entropylab.sh
EOF
chmod +x /Users/$USER_NAME/Self-Destruct.command
chown $USER_NAME:staff /Users/$USER_NAME/Self-Destruct.command

# Dock Pinning
sudo -u $USER_NAME defaults write com.apple.dock persistent-others -array-add "<dict><key>tile-data</key><dict><key>file-data</key><dict><key>_CFURLString</key><string>file:///Users/$USER_NAME/Self-Destruct.command</string><key>_CFURLStringType</key><integer>15</integer></dict></dict><key>tile-type</key><string>file-tile</string></dict>"

echo -e "${GREEN}✅ Self-destruct mechanism deployed.${NC}"

echo -e "\n${BLUE}======================================================================"
echo -e "🎉 PROVISIONING COMPLETE"
echo -e "======================================================================${NC}"
echo -e "1. Log out of Admin account."
echo -e "2. Log into 'entropylab' with the password you provided."
echo -e "3. Your hardened environment is ready."
