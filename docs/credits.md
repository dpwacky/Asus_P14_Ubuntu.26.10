# Credits

Credit for this image goes to Ubuntu, and to the two repository holders
whose laptop code it stands on. There is no personal credit here.

The remaster itself was done with existing knowledge of the machine and
with vibe coding. The sessions ran in Cursor, in particular with Grok and
Claude. ChatGPT has a separate place in that work: it was part of working
out the machine, alongside that knowledge and the vibe-coding sessions.

The laptop-specific code is upstream work, used here at the commits below.
Those repositories stay the source of truth.

## Ubuntu

The desktop, the installer, and the kernel base are Ubuntu 26.10 arm64.

- Desktop ISO: the official Ubuntu 26.10 desktop image for arm64, from
  [cdimage.ubuntu.com](https://cdimage.ubuntu.com/releases/26.10/release/).
  The file on that page is the stock image. This project publishes a
  remaster, not that original file.
- Kernel source package `linux` `7.3.0-9.9`, which is 7.3-rc6 plus Ubuntu's
  packaging. Launchpad: the Ubuntu `linux` source for 26.10.

Stock `linux-image-7.3.0-9-generic` does not bring up the ProArt P14 on its
own. The pieces below are what close that gap.

## Canonical and NVIDIA, the platform kernel

The N1X laptop enablement (MediaTek MT8901 peripherals, the GPU IOMMU, the
audio and power pieces) was written for Canonical's NVIDIA kernel tree.
Omarchy carries that series forward. Josephine names the base as NV-Kernels
`2234cd7f8f8a9fdbceef37f82c531016aab8cf73`.

This image rebases that series onto Ubuntu `7.3.0-9.9`. Those drivers were
not rewritten here.

## Omarchy

Omarchy is the project that already runs this class of machine, including
the ProArt P14. The boot contract and the memory fix come from that tree.
The holder of these repositories is David Heinemeier Hansson, under the
omacom organisation.

| Tree | Public repository | Commit used |
| --- | --- | --- |
| Installer and hardware recipes | [github.com/omacom/omarchy](https://github.com/omacom/omarchy) | `0a7beb7` |
| Package and kernel recipe | [github.com/omacom/omarchy-pkgs](https://github.com/omacom/omarchy-pkgs) | `c1e2d5e` |
| Their own ISO builder | [github.com/omacom/omarchy-iso](https://github.com/omacom/omarchy-iso) | same clone as the tree above |

What this image uses from those trees:

- The N1X hardware script,
  [install/hardware/n1x.sh](https://github.com/omacom/omarchy/blob/0a7beb7/install/hardware/n1x.sh):
  `console=tty0`, `acpi=nospcr`, `mem_sleep_default=s2idle`,
  `iommu.passthrough=1`, `power_wrap_drv.usb4_release=0`, the PCI hotplug
  window, and `initcall_blacklist=simpledrm_platform_driver_init`.
- The kernel recipe `linux-omarchy-aarch64` in omarchy-pkgs: the N1X patch
  series and the kernel options `CONFIG_MTK_POWER_WRAP`,
  `CONFIG_MTK_SSPM_CI`, and `CONFIG_IOMMU_DEFAULT_PASSTHROUGH`.
- The firmware GPU reservation, written as the EFI variable
  `EXPOSED_DISP_FRM`. Their script is
  [bin/omarchy-apply-n1x-gpu-memory](https://github.com/omacom/omarchy/blob/0a7beb7/bin/omarchy-apply-n1x-gpu-memory).
  The copy in this image is `/usr/local/sbin/n1x-gpu-memory`, same layout,
  4 GiB instead of the Windows default of about 62.5 GiB.

Omarchy is MIT licensed, copyright David Heinemeier Hansson. See
`LICENSE` in that repository.

## Josephine, ProArt P16

The second repository holder is Josephine (pfeifferj). Her image is for
the ASUS ProArt P16 (H7607BA), the larger sibling of this P14. Same N1X
platform.

- Repository:
  [github.com/pfeifferj/archlinuxarm-proart-p16](https://github.com/pfeifferj/archlinuxarm-proart-p16),
  commit `3484cc0`.
- The patch used here confines an ACPI table override to one EFI memory
  descriptor, so a table that crosses a boundary is still mapped. It is
  `0001-arm64-bound-acpi-override-allocation.patch` in her tree. The header
  credits the H7607A installer maintainers and bases it on NV-Kernels
  `2234cd7f`.

Her P16 work is where the SSPM / `_OSI("Windows 2022")` behaviour was
already written down. On this P14, BIOS .303 still has that check. The
command line denies the Windows 2022 string instead of shipping a DSDT
override.

## NVIDIA

- Open GPU kernel modules:
  [github.com/NVIDIA/open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules),
  tag `615.78.08`.
- The userspace libraries in the image are the matching 615.78.08 build
  (EGL, GBM allocator, NVDEC). They are not the Ubuntu archive packages
  for 615.71.09. Those archive packages are pinned off so an upgrade does
  not replace them.

## Cursor, Grok, Claude, and ChatGPT

The glue around those trees was written with vibe coding, on top of
knowledge of this laptop that was already there. That work happened in
Cursor. The models used there were Grok and Claude.

ChatGPT is called out on its own. It was part of figuring this machine
out, next to that knowledge and the Cursor sessions. It is not one of the
repository holders, and it is not Ubuntu. It still belongs in the credits.

The pieces that came out of that are GRUB entry 3, the live session that
lets the Ubuntu installer start, the kernel package
`linux-image-7.3.0-rc6-n1x` and the apt hold, the NVIDIA device nodes
before GDM, the touchpad quirk, and the UCM profile for the internal
speakers. Those are described in the other pages under `docs/`. The ISO
and the kernel build tree are not in git.
