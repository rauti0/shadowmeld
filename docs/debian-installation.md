# Debian Installation — ShadowMeld (NSM Sensor)

Headless install of Debian onto a Protectli VP2430 over a serial console.

## Hardware layout

- **eMMC** → operating system
- **NVMe SSD** → reserved for capture/pcap data
- **COM port** → serial console to workstation
- **Port 1 (enp1s0)** → management — DHCP via switch during install, later the workstation NAT link (static IP)
- **Port 4 (enp4s0)** → capture / mirror (no IP, listener only)

## 1. Create the installer USB

Download the current Debian netinst image from
<https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/> and write it to a
USB stick (verify the target device with `lsblk` first):

```bash
sudo dd if=debian-*-amd64-netinst.iso of=/dev/<USB_NAME> bs=4M status=progress oflag=sync
```

## 2. Connect serial console

On the workstation, open the serial console (115200 8N1).
Any serial terminal works with the same settings — (`picocom`) is used here:

```bash
picocom -b 115200 /dev/ttyUSB0
```

Power on the sensor. The firmware mirrors POST and the GRUB menu to serial.

## 3. Boot the installer over serial

At the installer's GRUB menu, highlight the text-mode **Install** entry and
press `e` to edit. Append a serial console to the `linux` line before `---`:

```
linux /install.amd/vmlinuz vga=788 console=ttyS0,115200n8 --- quiet
```

Boot the edited entry with `Ctrl-x`. The text installer now runs over serial.

> Serial console navigation: arrow keys may not pass through. Use
> `Ctrl-n` (down), `Ctrl-p` (up), `Ctrl-f` (forward), `Ctrl-e` (end of line).
> **Space** toggles a checkbox, **Enter** accepts the whole screen.

## 4. Installer choices

- **Network:** select the management NIC (enp1s0) as primary interface; it
  picks up an address via DHCP.
- **Root password:** leave blank → root account is locked, first user gets sudo.
- **User:** create a normal user (this account administers via `sudo`).

## 5. Partitioning

Target the **eMMC only** — leave the NVMe untouched.

- Guided partitioning, **use entire disk and set up LVM**
- Select the eMMC device (`mmcblk0`)
- "All files in one partition" is fine for a small OS disk
- Confirm the summary lists **only mmcblk0** (NVMe must not appear), write changes

A server scheme with a separate (`/var`) is the alternative — it keeps growing logs
and container images from filling the root partition. On a small 32 GB disk the
single-partition layout was simpler, and capture data lives on the NVMe anyway.

## 6. Software selection (tasksel)

Minimal headless host — **no desktop environment**:

```
[ ] Debian desktop environment
[ ] GNOME
[*] SSH server
[*] standard system utilities
```

Let the install finish and reboot.

## 7. First boot + remove media

Remove the USB stick **before** choosing Continue, so the sensor boots from the
eMMC rather than restarting the installer.

## 8. Make serial console persistent

The installed system does not redirect to serial automatically. On first boot,
edit the GRUB entry once (`e`, append `console=ttyS0,115200n8 console=tty0` to
the `linux` line, `Ctrl-x`) to reach the login.

```bash
linux /vmlinuz-6.12.90+deb13.1-amd64 root=/dev/mapper/sensor--vg-root ro quiet console=ttyS0,115200n8 console=tty0
```

(kernel version will vary)

Then make it permanent. Edit GRUB defaults:

```bash
sudoedit /etc/default/grub
```

Set the kernel command line (keeps both screen and serial usable; serial gets
the login prompt because it is listed last):

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet console=tty0 console=ttyS0,115200n8"
```

Add serial output for the GRUB menu itself:

```
GRUB_TERMINAL="console serial"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
```

Apply:

```bash
sudo update-grub
sudo reboot
```

After reboot the GRUB menu, full boot log, and login prompt all appear on the
serial console with no manual editing. Attaching a monitor/keyboard later also
works — both management paths are available in parallel.

## Serial parameters reference

`115200 8N1` = 115200 baud, 8 data bits, No parity, 1 stop bit. Both ends must
match. The setting lives on the Vault side (kernel + GRUB) and works with any
terminal program (picocom, tio, screen, minicom, PuTTY) set to the same values.
