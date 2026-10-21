# What the image changes, and why

The files under [overlay/](../overlay/) are the pieces added on top of the
Ubuntu 26.10 arm64 desktop image. A path in `overlay/` is the same path on
the installed system. The kernel source, the NVIDIA 615.78.08 libraries and
the ISO are not in this repository.

The image does not change the GNOME pointer speed. Mouse speed and touchpad
speed stay at `0`, which is the GNOME default
(`org.gnome.desktop.peripherals.mouse speed` and
`org.gnome.desktop.peripherals.touchpad speed`). A value set later in
Settings is stored in that user's dconf and is not part of the image.

## Touchpad

`overlay/etc/libinput/local-overrides.quirks`

The PixArt pad `093A:4F02` reports pressure as 0..65535. Real contacts sit
between 0 and a few hundred. libinput's default touch thresholds (30/25)
and palm threshold (130) then drop almost every contact, so the cursor does
not move even though the kernel delivers events. The quirk turns those
thresholds off for this vendor, product and DMI name. Same class of fix as
the ProArt P16 pad `093A:4F01`. GNOME applies it to a device it opens after
the file exists, which means a new login.

## Live GRUB menu

`overlay/boot/grub/grub.cfg`

BIOS .300 and .303 hide SSPM, COM1-3, SVP0 and HFRP when Linux claims
`_OSI("Windows 2022")`. Linux 6.12 and later answers that string, so the
SSPM mailbox never probes and the keyboard, touchpad, audio DSPs and USB4
host routers stay unbound. Every supported entry therefore carries the
literal `acpi_osi="!Windows 2022"`. A GRUB variable cannot hold that
argument: the space splits it and the quotes get escaped.

Option 3 is the desktop path: NVIDIA modeset and fbdev, and SimpleDRM kept
off so it does not sit in front of `nvidia-drm`. Option 6 leaves the
firmware `_OSI` behaviour in place on purpose, so the broken default can be
compared. The other arguments (`acpi=nospcr`, `mem_sleep_default=s2idle`,
`iommu.passthrough=1`, `power_wrap_drv.usb4_release=0`, the `pci=hpbus*`
room for docks) come from the Omarchy N1X boot line.

## Installed command line

`overlay/etc/default/grub.d/99-n1x.cfg`

The installed disk keeps the hardware line from option 3, including
`acpi_osi="!Windows 2022"`. The live-only masks (udev settle, device
timeouts, iio-sensor-proxy) stay off this file.

## GPU memory reservation

`overlay/usr/local/sbin/n1x-gpu-memory`
`overlay/etc/systemd/system/n1x-gpu-memory.service`

BIOS H7407BA.303 reserves about 62.5 GiB for a Windows GPU heap the open
driver does not use. The script writes the EFI variable `EXPOSED_DISP_FRM`
down to 4 GiB when the firmware layout matches, and leaves any other layout
alone. efivarfs rejects a shell redirect, so the write is one Python
`write()`. The new size applies on the next boot. The value lives in NVRAM
and survives a reinstall. See [memory.md](memory.md).

## NVIDIA device nodes and GDM

`overlay/etc/systemd/system/n1x-nvidia-nodes.service`
`overlay/etc/systemd/system/gdm.service.d/n1x-wait.conf`
`overlay/etc/modprobe.d/nvidia-n1x.conf`

GNOME only registers the session once `/dev/nvidiactl` exists. The oneshot
runs `nvidia-modprobe` before GDM and gives up after 25 seconds so a missing
GPU does not hold the boot. The GDM drop-in orders the display manager
after that job and after `/dev/dri/card0`. The modprobe file allows the open
modules on this GPU and keeps `nouveau` and `nvidiafb` from binding.
`NVreg_OpenRmEnableUnsupportedGpus=1` is required for PCI `10de:2e03`.

Modeset is chosen on the kernel command line, not in this file.

## Desktop user groups

`overlay/usr/local/sbin/n1x-live-session`
`overlay/etc/systemd/system/n1x-live-session.service`

`/dev/dri/renderD128` is mode 660 and group `render`. Without that group
NVIDIA hides the EGL device and the session stays on llvmpipe. On a live
boot the script also turns on GDM autologin for `ubuntu` and creates the
SSH user `n1x` (password `n1x2026`, sudo without a password). On an
installed system it only adds accounts in the normal uid range to
`render`, `video` and `input`.

## Firefox menu entry

`overlay/usr/local/share/applications/firefox_firefox.desktop`
`overlay/usr/share/applications/firefox-n1x.desktop`
`overlay/opt/firefox/distribution/policies.json`
`overlay/etc/environment`

Firefox 143 crashes on mutter 51's `wp_image_description_v1`, so the menu
entry starts `/opt/firefox` with `MOZ_ENABLE_WAYLAND=0` on that line only.
The variable is not in `/etc/environment`. Putting it there makes the snap
die as well: the snap forces `GDK_BACKEND=wayland` and then cannot open
display `:0`.

The snap also cannot load `libnvidia-rmapi-tegra`, so it draws with Mesa
and does not show up in `nvidia-smi`. The file under `usr/local/share` is
searched before the snap desktop and is the visible icon. `firefox-n1x.desktop`
is hidden so there is one icon.

`LIBVA_DRIVER_NAME=nvidia` selects `nvidia_drv_video.so`.
`MOZ_DISABLE_RDD_SANDBOX=1` lets the decoder process open the render node.
The policies lock hardware video decoding on. Playback is still not smooth.
See [desktop.md](desktop.md).

## Speakers

`overlay/usr/share/alsa/ucm2/MediaTek/mt8901-soundwir/`

The card name is truncated to `mt8901-soundwir`. PipeWire shows Dummy
Output until this UCM tree is installed. The P14 uses
`HiFi-cs42l43-cs35l56x2.conf` (CS42L43 and two CS35L56 on SoundWire link 1).
The profiles are NVIDIA's, under the BSD-3-Clause header in each file.
The `conf.d` symlink is how ALSA finds the card.

## Keyboard function keys

`overlay/etc/modprobe.d/hid_asus.conf`
`overlay/etc/modules-load.d/n1x-input.conf`

`hid_asus` has to be loaded for the internal keyboard `0B05:4B42`.
`fnlock_default=0` leaves F1-F12 as function keys, matching Windows, with
Fn-lock as the toggle. The modules file also loads `i2c_mt65xx` and
`i2c_hid_acpi`, which is the bus the keyboard and the touchpad sit on once
SSPM is visible.

## Windows NTFS volumes

`overlay/etc/modprobe.d/n1x-ntfs.conf`

The in-tree `ntfs` module rejects Windows LogFile 2.0. udisks and the
initrd request the name `ntfs`. A blacklist does not stop that request.
`install ntfs /sbin/modprobe ntfs3` hands it to `ntfs3`. The file has to be
inside the initrd, not only on the root filesystem.

## USB4 root ports

`overlay/etc/udev/rules.d/71-n1x-usb4-root-ports.rules`

The tunnel root ports `10de:22cf` cannot wake for a hotplug. The rule holds
them out of runtime suspend (`power/control=on`).

## Installer kernel package

`overlay/etc/apt/preferences.d/n1x-kernel`
`overlay/etc/apt/sources.list.d/n1x-local.sources`

The language layers still list the stock kernel. The N1X package is served
from `file:///cdrom/n1x-local` with origin N1X, so the pin (priority -1 for
`linux-image-*` and the Ubuntu NVIDIA packages from origin Ubuntu) does not
hide it. Apt then installs `linux-image-7.3.0-rc6-n1x` and does not pull
`linux-generic-hwe-26.04`. Skip the installer's offer to update itself.
A downloaded installer update can replace that choice.

## Initrd on a cross build

`overlay/etc/dracut.conf.d/10-n1x.conf`

The image is built on an x86_64 workstation. A host-only initrd would follow
that machine and miss the laptop keyboard, NVMe and USB. `hostonly` is off,
and those drivers are added explicitly.

## Live diagnostics and a shorter geoclue wait

`overlay/usr/local/sbin/n1x-logdump`
`overlay/etc/systemd/system/n1x-logdump.service`
`overlay/etc/systemd/system/n1x-logdump.timer`
`overlay/etc/systemd/system/geoclue.service.d/n1x-timeout.conf`
`overlay/etc/ssh/sshd_config.d/10-n1x-live.conf`

The log dump runs only when `/cdrom` exists, so an installed system does
not wait on it. Geoclue's start timeout is cut from 90 seconds to 10.
Password login over SSH is enabled for the live `n1x` account. That account
is not copied onto the installed system.

## VA-API driver package

[debs/nvidia-vaapi-driver_0.0.17-0n1x_arm64.deb](../debs/nvidia-vaapi-driver_0.0.17-0n1x_arm64.deb)

This file is not inside the ISO. The image carries unpackaged
`nvidia-vaapi-driver` 0.0.16, which cannot allocate decode surfaces on the
N1X. The package is upstream tag `v0.0.17`, cross-built on the 9950X3D2
workstation, and it replaces
`/usr/lib/aarch64-linux-gnu/dri/nvidia_drv_video.so` after install.
`sudo dpkg -i` is enough. It does not upgrade or replace
`linux-image-7.3.0-rc6-n1x`. See [debs/README.md](../debs/README.md).

The menu Firefox in `overlay/` is still 143 with `MOZ_ENABLE_WAYLAND=0`,
because that binary crashes as a Wayland client. Snaps do not receive
`libnvidia-rmapi-tegra`, so the store Firefox and Chromium stay on the CPU.
The working setup is Firefox 158 and the Chrome `.deb`, written up in
[video.md](video.md). Those two browsers are not files in `overlay/`.

## What is not in `overlay/`

The kernel is Ubuntu 7.3.0-9.9 plus the Omarchy N1X series plus Josephine's
ACPI allocation patch. The tree is large and is not published here. Credit
and the upstream links are in [credits.md](credits.md). The NVIDIA open
modules and userspace are 615.78.08. Do not replace them with Ubuntu's
`libnvidia-gl-615` (615.71.09).
