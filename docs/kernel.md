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

`linux-image-7.3.0-rc6-n1x-64m.2`, version `7.3.0-rc6.64m.3`.

`uname -r` after the reboot is `7.3.0-rc6-n1x-64m.2+`.

The package contains that kernel and the NVIDIA open modules 615.78.08
built for it (`nvidia`, `nvidia-modeset`, `nvidia-drm`, `nvidia-uvm`,
`nvidia-peermem`). It does not replace the 615.78.08 userspace libraries.
Installing it runs dracut and updates GRUB. The command line is the same
hardware line as the image kernel, including
`acpi_osi="!Windows 2022"` and `nvidia_drm.modeset=1`.

`7.3.0-rc6-n1x+` stays on disk and stays in the GRUB menu. This package
and the ISO kernel are held. An earlier build, `7.3.0-rc6-n1x-64m+`, also
stays if it was already installed. GRUB boots `7.3.0-rc6-n1x-64m.2+`
first. The apt pin that blocks `linux-generic` is unchanged.

The `.deb` is about 185 MB. GitHub rejects a file over 100 MB, so the
package is not in this repository. Download it after the ISO install:

[https://www.ponne.nu/linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb](https://www.ponne.nu/linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb)

SHA256 `1073589ef162f0196aca79253e9f65bd4a831b0d7b0ae49f519fb9c538c67239`.

On the installed system, after the first boot from the NVMe:

```sh
curl -fL -o linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb \
  https://www.ponne.nu/linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb
sha256sum linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb
sudo dpkg -i linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb
sudo reboot
```

After the reboot:

```sh
uname -r
grep CONFIG_ARCH_FORCE_MAX_ORDER= /boot/config-$(uname -r)
```

Expect `7.3.0-rc6-n1x-64m.2+` and `CONFIG_ARCH_FORCE_MAX_ORDER=14`.
`/proc/buddyinfo` then has fifteen counts on the Normal zone (orders 0
through 14). On the first minute of this boot the 64 MiB column held 1800
free blocks.

The 4K check was made on the previous 64 MiB kernel, `7.3.0-rc6-n1x-64m+`,
which uses the same order. One second of 3840x2160 H.264, decoded with
VA-API into VA-API frames, finished at about 3.4 times realtime. `dmesg`
had zero `NV_ERR_NO_MEMORY` lines. That is the allocation which used to
fail. On this laptop, 4K playback in Firefox 158 is smooth once a 64 MiB
kernel is running. 1440p was already smooth before it. This package does
not change that limit.

The ISO is unchanged. A fresh install still boots `7.3.0-rc6-n1x+` until
this package is installed. Do that after the first boot from the NVMe.

## USB4

The same package adds two Thunderbolt fixes from the Omarchy N1X series,
1085 and 1086. They apply to the NVIDIA host routers on this machine
(`NVDA8100`, one per USB-C port).

1085 keeps the tunnels, including DisplayPort, across system sleep when
the router is left awake. A resume otherwise tears those tunnels down.
A dock that never powered off sends no replug event, so its USB3 bandwidth
and its displays do not come back.

1086 waits 100 ms between DisplayPort tunnel teardowns. The module
parameter is `thunderbolt.dp_teardown_delay` and the default is 100. The
N1x GPU firmware misses one of two hotplug drops that arrive together, and
the second monitor then stays dark.

The command line already has `power_wrap_drv.usb4_release=0` and the PCI
hotplug padding. The udev rule in the image holds the `10de:22cf` tunnel
root ports out of runtime suspend.

Three `thunderbolt-platform NVDA8100: RX overflow for ring 0` lines are
still printed while the routers are probed. Those lines are not what 1085
and 1086 change. No dock was attached on this laptop, so sleep with a dock
and a second external monitor were not checked here.

## Fn-lock

`hid-asus` for the internal keyboard `0B05:4B42` loads from the initrd,
before `/etc/modprobe.d` is visible. This package puts
`options hid_asus fnlock_default=1` in
`/usr/lib/modprobe.d/hid-asus-p14.conf`, which the initrd does include.

`fnlock_default=1` is the state in which volume and the other working Fn
shortcuts are reported. Setting it to 0 was tried. Volume, brightness and
the rest of those shortcuts then stopped. A few Fn-row keys still do
nothing with the lock left on. The screen brightness keys are among them.
The GNOME slider changes the panel. Volume and the keyboard backlight work.
