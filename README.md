# Ubuntu 26.10 on the ASUS ProArt P14 (NVIDIA N1X), beta

This is a hobby project, and this release is a beta. I have run Ubuntu on
my workstations and servers for a long time, and I wanted the same thing
on this laptop: a normal Ubuntu desktop, on the disk, with the kernel I
actually boot. The installer and the desktop start. After the kernel package below, this
laptop runs as a normal Ubuntu machine. A few Fn keys still do nothing.
Read that limit before you wipe a disk you still need.

The machine is the ASUS ProArt P14 with the NVIDIA N1X, model H7407BA,
128 GB of memory. The image starts a GNOME session on the NVIDIA GPU and
can install Ubuntu onto the internal NVMe with that same kernel. It is not
an Ubuntu or NVIDIA product, and it is not a supported ASUS image. I built
it because I wanted the laptop to be one more Ubuntu machine.

The ISO is not stored in this repository. Download it from
[https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso](https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso).
The checksum file is next to it:
[https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso.sha256](https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso.sha256).
Credit for the code goes to Ubuntu and to the two
repository holders named in [docs/credits.md](docs/credits.md), and to
Cursor (Grok and Claude) and ChatGPT for the vibe-coding work. There is
no personal credit on that page.
How the ISO was built, and on which workstation, is in
[docs/build.md](docs/build.md). The files added on top of Ubuntu, and why
each one is there, are in [docs/changes.md](docs/changes.md). The files
themselves are in [overlay/](overlay/). Video acceleration, and why the
Firefox and Chromium snaps do not use the NVIDIA GPU, is in
[docs/video.md](docs/video.md). A later kernel that allows a 64 MiB
physically contiguous allocation is in [docs/kernel.md](docs/kernel.md).

## Update the BIOS to 303 first

Do this while Windows is still installed. This image was brought up on
BIOS **H7407BA.303**. Boot it on that firmware.

1. In Windows, install updates, including anything MyASUS offers for the
   ProArt P14.
2. Reboot until the firmware setup shows **H7407BA.303**.
3. Only then write this stick and install Ubuntu.

The BIOS update is how this model reaches .303. I did not find a separate
file to download and flash by hand. Wiping Windows first leaves you without
the path that delivered this firmware.

.303 still hides the SSPM mailbox when Linux claims to be Windows 2022.
Entry 3 turns that claim off. Older firmware was not what this image was
tested on.

## Hardware

Tested on one machine:

| Item | Value |
| --- | --- |
| Model | ASUS ProArt P14, H7407BA |
| BIOS | H7407BA.303 |
| CPU | 20 Arm cores (Cortex-X925 / A725) |
| Memory | 128 GB installed. Up to 112 GB works; 96 GB was tested |
| GPU | NVIDIA RTX Spark N1X, PCI `10de:2e03` |
| Wi-Fi / Bluetooth | MediaTek MT7925 |

## What works

On the live desktop, after GRUB option 3:

- The keyboard works.
- The keyboard backlight works.
- The trackpad works.
- The volume keys work.
- Volume works: internal speakers, the headphone jack and the microphone array.
- Power works. The power button works. Balanced and power-saver profiles work. There is no performance profile; the firmware does not offer one.
- Memory works up to 112 GB. 96 GB was tested. The laptop has 128 GB installed. Details are in [docs/memory.md](docs/memory.md).
- The touchscreen and the stylus work.
- Wi-Fi, Bluetooth and Ethernet on a USB adapter work.
- The webcam works.
- The GNOME desktop runs on the NVIDIA GPU. `gnome-shell` and Xwayland are GPU clients.
- The GNOME brightness slider changes the panel brightness.
- Firefox video uses the NVIDIA decoder after the steps in [docs/video.md](docs/video.md) and the kernel package below. On this laptop, 1440p and 4K playback in Firefox 158 are smooth once that package is installed. The flashed image still has the older decoder, Firefox 143 and the 4 MiB kernel, so those steps come after the install.

## What does not work yet

- A few Fn-row keys still do nothing. The screen brightness keys are among them. The GNOME brightness slider changes the panel. Volume and the keyboard backlight work. Setting `hid_asus fnlock_default=0` stops the shortcuts that do work, so the follow-up kernel leaves Fn-lock on.
- The Firefox snap and the Chromium snap do not load `libnvidia-rmapi-tegra`, so a browser installed from the store draws on the CPU. Use the Mozilla tarball and the Chrome `.deb` in [docs/video.md](docs/video.md). The store packages do not grow into the NVIDIA path on their own.

Firmware lines that remain in `dmesg` after a successful boot are listed in
[docs/firmware-messages.md](docs/firmware-messages.md).

## Image

| | |
| --- | --- |
| File | `ubuntu-26.10-desktop-arm64-n1x-final.iso` |
| Download | [https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso](https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso) |
| Checksum file | [https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso.sha256](https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso.sha256) |
| SHA256 | `846b9d4fd7e2c69305ce92a3ba66b5c32d025765382c64d9bc6f37b60e567756` |

| Piece | Version |
| --- | --- |
| Kernel on the ISO | `7.3.0-rc6-n1x+`, package `linux-image-7.3.0-rc6-n1x` |
| Later kernel, not in the ISO | `7.3.0-rc6-n1x-64m.2+`, package `linux-image-7.3.0-rc6-n1x-64m.2` version `7.3.0-rc6.64m.3`. 64 MiB contiguous blocks and the USB4 sleep and DisplayPort fixes. See [docs/kernel.md](docs/kernel.md) |
| Base | Ubuntu 26.10 (7.3.0-9.9) plus the N1X platform patches |
| NVIDIA open modules and userspace | 615.78.08 |
| Desktop | Ubuntu 26.10 GNOME |

Check the download before writing it:

```sh
sha256sum -c ubuntu-26.10-desktop-arm64-n1x-final.iso.sha256
```

Run that in the directory where both downloaded files sit. `sha256sum` prints `OK` when the ISO matches.

## Write the USB stick

Replace `/dev/sdX` with the stick. The whole device is overwritten.

```sh
sudo dd if=ubuntu-26.10-desktop-arm64-n1x-final.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

Confirm the target with `lsblk` first. A wrong disk name erases that disk.

## Boot the installer

The kernel on this stick is unsigned. Secure Boot has to be off, and the
stick has to be chosen from the firmware boot menu. Hold the key as soon
as you press the power button.

1. Hold **F2** while the laptop starts. That opens the BIOS setup.
   Set **Secure Boot** to **Disabled**. Save and exit.
   Leave it disabled after the install as well. The N1X kernel is not
   signed, so Secure Boot would stop the system that lands on the NVMe.
2. Hold **ESC** while the laptop starts. The firmware boot menu opens.
   Choose **EFI USB disk** (the stick you just wrote). That is the
   installer. The internal NVMe entry is the disk you already have, not
   this image.
3. In the GRUB menu, choose option **3**, "desktop on the NVIDIA GPU".

Option 3 is the supported path. Option 6 drops the firmware workaround on
purpose and is only there for comparison. The keyboard and the speakers
depend on that workaround. See [docs/boot.md](docs/boot.md).

The live session logs in as `ubuntu` with an empty password. The Ubuntu
installer is the normal desktop installer and should open on its own.

SSH on the stick, if you need it: user `n1x`, password `n1x2026`, sudo
without a password. That account is created while the stick is running. It
is not copied onto the installed system.

## This image writes an EFI variable, and the value stays

Booting this image writes the EFI variable `EXPOSED_DISP_FRM`. BIOS
H7407BA.303 reserves about 62.5 GiB for a Windows GPU heap. When the
firmware layout matches, the image sets that reservation to 4 GiB. The new
size takes effect on the next boot, and usable memory then goes up to
112 GB.

The variable is stored in NVRAM, not on the disk. Wiping the NVMe,
reinstalling Ubuntu, or putting Windows back does not restore the 62.5 GiB
value. The laptop keeps the 4 GiB reservation until something writes the
variable again. Read this before you install. The layout and the script are
in [docs/memory.md](docs/memory.md).

## Install onto the NVMe

The installer copies the live system onto the disk you choose and then
boots the N1X kernel from that disk. Hold F2 and disable Secure Boot, hold
ESC and choose EFI USB disk, then GRUB option 3. Let the installer open.

When the installer asks to update itself, choose **Skip**. That prompt is
easy to miss and it matters. The installer on this image already knows the
N1X kernel. A downloaded installer update can replace that and pull the
stock Ubuntu kernel instead.

Then use these choices. This is the path that was used on this laptop:

1. **Interactive installation.**
2. **Default selection** (the smaller app set). Extended selection is the
   full desktop on the same root and the same kernel. Default is what was
   used here.
3. Leave **Install third-party software for Wi-Fi and graphics**
   unchecked. NVIDIA 615.78.08 is already in the image. The Ubuntu archive
   driver is pinned so apt will not replace it.
4. **Download and install support for additional media formats** is
   optional. Either choice is fine.
5. **Erase disk and install Ubuntu** on the internal NVMe. This removes
   the partitions that are on it, including a Windows installation.
6. **No encryption.** Encryption might work. It was not tested.
7. Fill in the rest as usual: your name, the computer name, the username
   and the password, then the time zone.

Finish the install and reboot from the NVMe. Remove the USB stick when the
firmware starts again. Secure Boot stays disabled.

The installed GRUB command line is the hardware line from
[docs/boot.md](docs/boot.md), including `acpi_osi="!Windows 2022"`. It does
not include the live-only masks.

The first account you create is added to the `render`, `video` and `input`
groups on the next boot. The GPU device nodes are mode `660` and group
`render`.

## After the first boot, install the 64 MiB kernel

The ISO kernel is `7.3.0-rc6-n1x+`. Its largest physically contiguous
block is 4 MiB, so a 4K video plane (about 8 MiB) and other single DMA
buffers of that size fail. Install the follow-up package on the installed
system, then reboot. It keeps the 64 MiB blocks, and it adds the USB4
fixes in [docs/kernel.md](docs/kernel.md). On this laptop, 4K playback in
Firefox is smooth after that reboot.

Download:

[https://www.ponne.nu/linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb](https://www.ponne.nu/linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb)

SHA256 `1073589ef162f0196aca79253e9f65bd4a831b0d7b0ae49f519fb9c538c67239`.

```sh
curl -fL -o linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb \
  https://www.ponne.nu/linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb
sha256sum linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb
sudo dpkg -i linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb
sudo reboot
```

`sha256sum` prints the hash above when the download matches. After the
reboot, `uname -r` prints `7.3.0-rc6-n1x-64m.2+`. The ISO kernel stays in
the GRUB menu. An earlier 64 MiB package, if it was already installed,
stays there too. Why this package exists is in [docs/kernel.md](docs/kernel.md).

## The kernel stays put

`linux-image-7.3.0-rc6-n1x` is held. Packages named `linux-image-*`,
`linux-headers-*`, `linux-modules-*` and `linux-generic*` from the Ubuntu
archive are pinned to priority -1. Unattended upgrades skip those names and
the Ubuntu NVIDIA driver packages.

`apt update` and `apt upgrade` still apply other updates. They do not
replace this kernel with `linux-generic-hwe-26.04`.

Check after the first boot from the NVMe:

```sh
uname -r
apt-mark showhold
```

`uname -r` on a fresh install should print `7.3.0-rc6-n1x+`.

That kernel is held, and it is enough to boot and to run the desktop. It
cannot hand out one physically contiguous block larger than 4 MiB. A 4K
video plane is about 8 MiB, and the same ceiling applies to any other
single DMA buffer of that size. Ordinary CUDA allocations are not one
block, and they already worked. The follow-up package
`linux-image-7.3.0-rc6-n1x-64m.2` raises the limit to 64 MiB, adds the
USB4 fixes, and leaves the original kernel in the GRUB menu. Download it
from
[ponne.nu](https://www.ponne.nu/linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb)
and install it with the commands above. The file is about 185 MB, past
GitHub's limit, so it is not stored in this repository. The checksum and
the reason are in [docs/kernel.md](docs/kernel.md).

Video decode still needs the userspace pieces: the 0.0.17 VA-API library,
Firefox 158 from Mozilla's tarball, and Google Chrome from its `.deb`. The
snaps do not see the NVIDIA library. Those steps are in
[docs/video.md](docs/video.md).

## Memory

The firmware ships a large GPU reservation (about 62.5 GiB). This image
writes the EFI variable `EXPOSED_DISP_FRM` down to 4 GiB when it finds the
known layout. The change applies on the next boot. Usable memory then goes
up to 112 GB. 96 GB of that was tested. CUDA takes from the same pool, with
the 64-bit calls (`cuMemAlloc_v2` and the matching memset, copy, free and
context calls).

The variable lives in NVRAM, not in the root filesystem. A reinstall does
not put the 62.5 GiB reservation back. A machine that already has the 4 GiB
value keeps it. Details are in [docs/memory.md](docs/memory.md).

## Credits and how to build it

Credit goes to Ubuntu, to the Omarchy repositories (David Heinemeier
Hansson, omacom), and to Josephine's
[Arch Linux ARM image for the ProArt P16](https://github.com/pfeifferj/archlinuxarm-proart-p16).
The GPU modules are
[NVIDIA's open kernel modules](https://github.com/NVIDIA/open-gpu-kernel-modules)
at 615.78.08. The remaster applied existing knowledge of this laptop and
vibe coding in Cursor, with Grok and Claude, and a separate part of the
work in ChatGPT. Full links and commits are in
[docs/credits.md](docs/credits.md).

Rebuilding the ISO wants an x86_64 Ubuntu workstation with a cross compiler,
a lot of disk, and the upstream trees above. The machine I used is an ASUS
ROG STRIX X870-F with a Ryzen 9 9950X3D2, 96 GB of RAM and Ubuntu 24.04.5.
The steps are in [docs/build.md](docs/build.md).

## More detail

- [Credits and upstream code](docs/credits.md)
- [Build the ISO](docs/build.md)
- [Boot command line](docs/boot.md)
- [Desktop and GPU](docs/desktop.md)
- [Video acceleration](docs/video.md)
- [64 MiB contiguous kernel](docs/kernel.md)
- [Memory](docs/memory.md)
- [What the installer keeps](docs/install.md)
- [Firmware messages that stay in dmesg](docs/firmware-messages.md)
- [What the image changes, and the files](docs/changes.md)
- [Packages to add after install](debs/README.md)
