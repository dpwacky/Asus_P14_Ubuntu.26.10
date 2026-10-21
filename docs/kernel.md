# A kernel that can hand out a 64 MiB block

The image still installs `7.3.0-rc6-n1x+`. That kernel is the right one for
this laptop, and it stays installed. It has one limit that shows up as a
GPU out-of-memory error even when tens of gigabytes are free.

## What failed

On 4 KiB pages the page allocator's largest physically contiguous block is
`CONFIG_ARCH_FORCE_MAX_ORDER`. The image kernel sets that to 10, which is
4 MiB. `nvidia-vaapi-driver` 0.0.17 asks the open NVIDIA driver for one
contiguous system-memory block per frame plane (`NV01_MEMORY_SYSTEM`,
big pages, physically contiguous). A 1440p luma plane is about 3.75 MiB and
fits. A 4K luma plane is 3840 by 2176 bytes, about 8.0 MiB, and does not.
The driver then logs `NV_ERR_NO_MEMORY` from `system_mem.c`. RAM is not
exhausted. `nvidia-smi` has no VRAM counter on this GPU, so the message
looks like the card is full.

CUDA and most game allocations do not take this path. `cuMemAlloc_v2`
builds a mapping out of ordinary pages. 16 GiB and 90 GiB were already
tested on `7.3.0-rc6-n1x+`. What stops at 4 MiB is a single physical chunk:
a block-linear video surface, and any other DMA or scanout buffer that the
driver refuses to split.

ARM64 with 4 KiB pages allows this order up to 15 (128 MiB). The Kconfig
text says to raise it when a large contiguous allocation is required. This
tree sets the 4 KiB default to 14, a 64 MiB block. In this kernel the
page-block size follows the same order, so the allocator also tracks free
memory in 64 MiB chunks. On 128 GB that is acceptable. The desktop, the
NVIDIA modules and the previous boot arguments came up on the first reboot.

## The package

`linux-image-7.3.0-rc6-n1x-64m`, version `7.3.0-rc6.64m`.

`uname -r` after the reboot is `7.3.0-rc6-n1x-64m+`.

The package contains that kernel and the NVIDIA open modules 615.78.08
built for it (`nvidia`, `nvidia-modeset`, `nvidia-drm`, `nvidia-uvm`,
`nvidia-peermem`). It does not replace the 615.78.08 userspace libraries.
Installing it runs dracut and updates GRUB. The command line is the same
hardware line as the image kernel, including
`acpi_osi="!Windows 2022"` and `nvidia_drm.modeset=1`.

`7.3.0-rc6-n1x+` stays on disk and stays in the GRUB menu. Both image
packages are held. The apt pin that blocks `linux-generic` is unchanged.

The `.deb` is about 185 MB. GitHub rejects a file over 100 MB, so the
package is not in this repository. The file that was installed on this
laptop is:

```
linux-image-7.3.0-rc6-n1x-64m_7.3.0-rc6.64m_arm64.deb
```

SHA256 `c8c04ab5bb57d50919b1deab12f8badb1123efdb7e63b99d18756e370d157e19`.

On an installed system, from the directory that contains that file:

```sh
sudo dpkg -i linux-image-7.3.0-rc6-n1x-64m_7.3.0-rc6.64m_arm64.deb
sudo reboot
```

After the reboot:

```sh
uname -r
grep CONFIG_ARCH_FORCE_MAX_ORDER= /boot/config-$(uname -r)
```

Expect `7.3.0-rc6-n1x-64m+` and `CONFIG_ARCH_FORCE_MAX_ORDER=14`.
`/proc/buddyinfo` then has fifteen counts on the Normal zone (orders 0
through 14). On the first minute of this boot the 64 MiB column held 1782
free blocks.

Checked on this laptop after that reboot: one second of 3840x2160 H.264,
decoded with VA-API into VA-API frames, finished at about 3.4 times
realtime. `dmesg` had zero `NV_ERR_NO_MEMORY` lines. That is the allocation
which used to fail. It does not by itself say that Firefox 4K playback is
smooth. Firefox can still spend CPU in the RDD process moving frames. That
cost was measured on the previous kernel and has not been remeasured in the
browser on this one. 1440p in Firefox 158 was already smooth before this
package.

The ISO is unchanged. A fresh install still boots `7.3.0-rc6-n1x+` until
this package is added.
