# Install

The install image uses the stock Ubuntu desktop installer. The live session
logs in as `ubuntu`. Snapd is running, so the installer snap can start.

The root that gets copied already contains the N1X kernel files.
`/etc/subiquity/kernel-meta-package` names `linux-image-7.3.0-rc6-n1x`, so
the installer asks apt for that package and does not pull
`linux-generic-hwe-26.04`. The package itself is on the ISO under
`n1x-local/`, as a local apt repository with origin N1X. The language
layers still carry the stock kernel in their package database, so apt has
to install this package from the disc and then drop the stock kernel.
The package is held. Archive packages `linux-image-*`, `linux-headers-*`,
`linux-modules-*` and `linux-generic*` are pinned to priority -1.
Unattended upgrades skip those names and the Ubuntu NVIDIA driver
packages.

## Choices in the installer

When the installer asks to update itself, choose **Skip**. The installer
shipped on this image already points at `linux-image-7.3.0-rc6-n1x`.
Updating it from the network can replace that and install the stock
Ubuntu kernel.

The path used on this laptop:

- Interactive installation
- Default selection
- Leave "Install third-party software for Wi-Fi and graphics" unchecked.
  NVIDIA 615.78.08 is already installed, and the archive driver packages
  are pinned.
- "Download and install support for additional media formats" is optional
- Erase disk and install Ubuntu
- No disk encryption. That option might work. It was not tested.
- Name, computer name, username and password are filled in as usual

The installed GRUB command line is the entry 3 line in
[boot.md](boot.md), including the literal `acpi_osi="!Windows 2022"`.
The initrd on the installed disk needs the keyboard modules for a LUKS
prompt, and the NTFS redirect so the Windows LogFile 2.0 volumes are not
claimed by the old `ntfs` driver:

```
install ntfs /sbin/modprobe ntfs3
blacklist ntfs
```

A blacklist alone does not stop an explicit `modprobe ntfs` from the
initrd. The file has to be inside the initrd, not only on the root
filesystem. The messages appear at about 2.6 seconds.

## Keep on the installed disk

- `n1x-nvidia-nodes.service` and `gdm.service.d/n1x-wait.conf`
- `/usr/bin/nvidia-modprobe` and `/usr/lib/udev/rules.d/71-nvidia.rules`
- `/etc/modprobe.d/n1x-ntfs.conf`
- `n1x-gpu-memory.service` and `/usr/local/sbin/n1x-gpu-memory`
- NVIDIA 615.78.08 userspace, including the sonames in
  [desktop.md](desktop.md), `ld.so.cache`, and `dri/nvidia_drv_video.so`
- the MT8901 UCM tree and the PixArt libinput quirk
- Cirrus `cs35l56-b0-dsp1-misc-104333a1` firmware
- the desktop user in group `render`
- `/etc/environment` entries `LIBVA_DRIVER_NAME=nvidia`,
  `MOZ_ENABLE_WAYLAND=0`, `MOZ_DISABLE_RDD_SANDBOX=1`
- Firefox under `/opt/firefox` if that browser is how video decode is shipped

The 4 GiB GPU reservation is an EFI variable in NVRAM. It is already set on
the machine this image was built for. A new machine needs
`n1x-gpu-memory.service` once, then one reboot.

## Live-only behaviour

These stay on the stick and do not define the installed system:

- `n1x-live-session` turns on GDM autologin for `ubuntu` and creates the
  SSH user `n1x` only when `/etc/casper.conf` exists
- `n1x-logdump.timer` runs only when `/cdrom` exists
- `casper-md5check.service` is masked
- anacron is masked on the live root

The installed GDM config has automatic login turned off. The account
created by the installer is the one that logs in.

The squashfs for this image is built with `-all-root` and the two pseudo
files `pseudo-owner.txt` and `pseudo-xattr.txt`. An unprivileged unpack
lands as uid 1000, with no setuid bit on `nvidia-modprobe` and no file
capabilities.
