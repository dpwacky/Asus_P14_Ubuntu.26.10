# Build the ISO

This is the workstation and the outline I used. It is a remaster of the
Ubuntu 26.10 arm64 desktop ISO, not a from-scratch distribution. The ISO
and the multi-gigabyte build tree are not in this repository. The
published file is
[https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso](https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso),
with its checksum at
[https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso.sha256](https://www.ponne.nu/ubuntu-26.10-desktop-arm64-n1x-final.iso.sha256).
The upstream
code is listed in [credits.md](credits.md). Clone those repositories at the
commits named there.

## Workstation

The image was built on my desktop, not on the laptop.

| Item | Value |
| --- | --- |
| Board | ASUS ROG STRIX X870-F GAMING WIFI |
| CPU | AMD Ryzen 9 9950X3D, 16 cores / 32 threads |
| Memory | 96 GB |
| Disk | about 1.8 TB, with a few hundred GB free while building |
| OS | Ubuntu 24.04.5 LTS, x86_64 |
| Kernel on that machine | 7.2.6 generic |

A smaller machine can do it. Cross-compiling the arm64 kernel and compressing
the squashfs is what wants the cores and the disk. Plan for at least 150 GB
free. The kernel tree, the unpacked squashfs and the ISO each take tens of
GB.

## Packages on the build machine

Ubuntu 24.04 names:

- `build-essential`, `bc`, `bison`, `flex`, `libelf-dev`, `libssl-dev`,
  `libncurses-dev`
- `gcc-aarch64-linux-gnu` (the kernel and the NVIDIA modules are built for
  arm64 on this x86_64 machine)
- `squashfs-tools`, `xorriso`
- `dpkg-dev`, `rsync`
- `qemu-user-static` if you configure the arm64 root with `dpkg` inside a
  chroot

The laptop does not need a compiler. Nothing in the image is built on it.

## What you start from

1. The official Ubuntu 26.10 desktop arm64 ISO. See
   [credits.md](credits.md).
2. Ubuntu kernel source `linux` `7.3.0-9.9`.
3. The Omarchy N1X patch series from
   [omarchy-pkgs](https://github.com/omacom/omarchy-pkgs), recipe
   `linux-omarchy-aarch64`, at commit `c1e2d5e`.
4. Josephine's ACPI allocation patch from
   [archlinuxarm-proart-p16](https://github.com/pfeifferj/archlinuxarm-proart-p16)
   at commit `3484cc0`.
5. [NVIDIA open GPU kernel modules](https://github.com/NVIDIA/open-gpu-kernel-modules)
   tag `615.78.08`, built against the kernel from steps 2 to 4.
6. The matching NVIDIA 615.78.08 userspace libraries.

Kernel options that have to be built in, taken from the Omarchy recipe:
`CONFIG_MTK_POWER_WRAP=y`, `CONFIG_MTK_SSPM_CI=y`, and
`CONFIG_IOMMU_DEFAULT_PASSTHROUGH=y`. The local version string on this
image is `-n1x+`, so `uname -r` is `7.3.0-rc6-n1x+`.

## Shape of the remaster

The stock ISO is unpacked. The root squashfs (`casper/minimal.squashfs`)
is where the laptop-specific files go. The live layer
(`casper/minimal.standard.live.squashfs`) stays the Ubuntu one, so snapd
and the desktop installer keep working.

Into the root:

- `/boot/vmlinuz-7.3.0-rc6-n1x+`, `/lib/modules/7.3.0-rc6-n1x+`, and an
  initrd that contains the NVMe driver, the I2C keyboard modules, and
  `/etc/modprobe.d/n1x-ntfs.conf`.
- The kernel packaged as `linux-image-7.3.0-rc6-n1x` and marked hold.
  `/etc/subiquity/kernel-meta-package` contains that package name, so the
  installer does not ask for `linux-generic-hwe-26.04`.
- `/etc/apt/preferences.d/n1x-kernel` pins archive kernels and the Ubuntu
  NVIDIA driver packages to priority -1.
- `/etc/default/grub.d/99-n1x.cfg` with the installed command line from
  [boot.md](boot.md).
- NVIDIA 615.78.08 userspace, `nvidia-modprobe`, the udev rule, and
  `n1x-nvidia-nodes.service` so `/dev/nvidiactl` exists before GDM.
- The UCM profile for the internal speakers, the PixArt touchpad quirk,
  and `n1x-gpu-memory.service` for the 4 GiB firmware reservation.

GRUB entry 3 on the stick uses the same hardware command line and leaves
snapd running. The live session logs in as `ubuntu`. The account `n1x` is
created only when `/etc/casper.conf` is present.

Repack the root with:

```sh
mksquashfs squash/root casper/minimal.squashfs \
  -comp xz -b 131072 -Xbcj arm -noappend -no-exports -all-root \
  -pseudo-override -pf pseudo-owner.txt -pf pseudo-xattr.txt
```

`-all-root` matters. An unsquashfs done as a normal user stores uid 1000
and drops the setuid bit on `nvidia-modprobe`. The two pseudo files put
the ownership and the file capabilities back.

Write a new hybrid ISO from the previous one so the Ubuntu boot catalog
stays intact:

```sh
xorriso -indev previous.iso -outdev new.iso \
  -boot_image any replay \
  -volid 'Ubuntu 26.10 N1X arm64' \
  -map grub.cfg /boot/grub/grub.cfg \
  -map md5sum.txt /md5sum.txt \
  -map minimal.squashfs /casper/minimal.squashfs \
  -map minimal.size /casper/minimal.size \
  -map install-sources.yaml /casper/install-sources.yaml
```

`xorriso` refuses to overwrite an existing `-outdev`. Update `md5sum.txt`
for every file you replace, and set `kernel.default` in
`install-sources.yaml` to `linux-image-7.3.0-rc6-n1x`.

Then check the SHA256 and boot entry 3 on a machine that is already on
BIOS H7407BA.303.
