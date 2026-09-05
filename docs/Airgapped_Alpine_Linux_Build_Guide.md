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
  find /tmp/modloop -type f \\( \
    -name 'modules.dep*' -o -name 'modules.alias*' -o \
    -name 'modules.symbols*' -o -name 'modules.builtin*' -o \
    -name 'modules.devname' -o -name 'modules.softdep' \
  \\) -delete && \
  mksquashfs /tmp/modloop /modloop_temp -noappend -comp xz
"

# Copy the modified modloop back to the original location
cp "$MODLOOP_TEMP" "$MODLOOP"
rm "$MODLOOP_TEMP"
echo "✅ modloop rebuilt and hardened."
```
