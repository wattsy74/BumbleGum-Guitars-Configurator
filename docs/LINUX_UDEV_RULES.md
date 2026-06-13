# Linux USB Device Access - udev Rules Configuration

## Overview

On Linux systems, direct access to USB devices requires appropriate permissions. The BumbleGum Guitars Configurator communicates with the BGG USB HID controller via serial communication (CDC - Communications Device Class), which requires proper udev rules to grant user-level access without requiring root/sudo privileges.

## Why udev Rules?

By default, USB devices are owned by the `root` user with limited permissions. Without proper udev rules:
- Regular users cannot access the BGG controller
- The application would need to run with `sudo` (not recommended)
- Permission issues would prevent normal operation

## Installation

### Step 1: Create the udev Rule File

Create a new file at `/etc/udev/rules.d/99-bgg-controller.rules`:

```bash
sudo nano /etc/udev/rules.d/99-bgg-controller.rules
```

### Step 2: Add the Following Rules

```udev
# BumbleGum Guitars Controller USB Rules
# Allows non-root users to access the BGG USB HID controller

# Generic CDC (Communications Device Class) Rule for BGG Controller
SUBSYSTEM=="tty", ATTRS{idVendor}=="2e8a", ATTRS{idProduct}=="000a", MODE="0666", GROUP="dialout"

# Alternative: Match by interface description (if the above doesn't work)
SUBSYSTEM=="tty", ATTRS{interface}=="*BGG*", MODE="0666", GROUP="dialout"

# USB device rule (lower-level access)
SUBSYSTEM=="usb", ATTRS{idVendor}=="2e8a", ATTRS{idProduct}=="000a", MODE="0666", GROUP="dialout"

# Serial Port Rule (catch-all for serial devices)
SUBSYSTEM=="ttyUSB*", ATTRS{idVendor}=="2e8a", MODE="0666", GROUP="dialout"
SUBSYSTEM=="ttyACM*", ATTRS{idVendor}=="2e8a", MODE="0666", GROUP="dialout"
```

**Note:** Replace `2e8a` and `000a` with your BGG controller's actual Vendor ID and Product ID if different.

### Step 3: Reload udev Rules

After creating the rules file, reload the udev daemon:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### Step 4: Verify the Rules

Disconnect and reconnect your BGG controller, then verify permissions:

```bash
ls -la /dev/ttyUSB* /dev/ttyACM*
```

You should see permissions like `crw-rw-rw-` (666) instead of `crw-rw----` (660).

## Finding Your Device's Vendor and Product IDs

If the above rules don't work, find your specific device IDs:

```bash
lsusb
```

Look for the BGG controller in the output. It will show:
```
Bus 001 Device 005: ID 2e8a:000a Raspberry Pi Pico
```

The format is `ID VENDOR_ID:PRODUCT_ID`.

Alternatively, while connected, check:

```bash
udevadm info -a -p $(udevadm trigger --dry-run | grep -i serial)
```

## Verify with dmesg

To see if your device is being recognized:

```bash
sudo dmesg | tail -20
```

You should see CDC messages when connecting the device.

## Troubleshooting

### Device not showing up
1. **Check connection:** `lsusb` should list your device
2. **Check kernel messages:** `sudo dmesg | grep -i usb`
3. **Verify rules:** `sudo udevadm test $(udevadm trigger --dry-run)`
4. **Reload rules again:**
   ```bash
   sudo udevadm control --reload-rules
   sudo udevadm trigger
   ```

### Permission still denied
1. Ensure your user is in the `dialout` group:
   ```bash
   groups $USER
   ```
2. If not listed, add your user:
   ```bash
   sudo usermod -a -G dialout $USER
   ```
3. Log out and log back in for group changes to take effect

### Device works with sudo but not as regular user
1. Verify the udev rule matches your device:
   ```bash
   sudo udevadm info -a -p /sys/bus/usb/devices/*/
   ```
2. Check that MODE is set to `0666` (world-readable/writable)
3. Ensure GROUP is set to `dialout` or another group your user belongs to

## Automated Installation (Optional)

For deployment, you can create an installation script that sets up the udev rules automatically:

```bash
#!/bin/bash
# install-udev-rules.sh

sudo cp 99-bgg-controller.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo usermod -a -G dialout $USER

echo "udev rules installed. Please log out and back in for group changes to take effect."
```

## Post-Installation Instructions for Users

Include these instructions in the Linux release documentation:

1. **First-time setup:**
   ```bash
   sudo udevadm control --reload-rules
   sudo udevadm trigger
   sudo usermod -a -G dialout $USER
   ```
   Log out and back in.

2. **Connect BGG controller**
3. **Run the application** - no sudo required

## References

- [udev Rules Documentation](https://man7.org/linux/man-pages/man7/udev.7.html)
- [Electron Serial Port Permissions](https://github.com/serialport/node-serialport/wiki/Debugging#linux)
- [Linux USB CDC Class](https://www.usb.org/class-code-information-center)
