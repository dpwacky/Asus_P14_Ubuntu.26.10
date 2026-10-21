# Packages to install after Ubuntu is on the NVMe

The ISO already boots kernel `7.3.0-rc6-n1x+`. The VA-API package below
does not replace that kernel. `apt upgrade` still will not pull
`linux-generic`, because of the pin in
`overlay/etc/apt/preferences.d/n1x-kernel`.

A second kernel, `linux-image-7.3.0-rc6-n1x-64m.2` version
`7.3.0-rc6.64m.3`, raises the largest physically contiguous allocation
from 4 MiB to 64 MiB and adds the USB4 sleep and DisplayPort teardown
fixes. Install it after the ISO is on the NVMe. The `.deb` is about 185 MB,
over GitHub's file limit, so it is not in this directory. Download it from
[https://www.ponne.nu/linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb](https://www.ponne.nu/linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb).
The SHA256 and the reason the image kernel was not enough are in
[docs/kernel.md](../docs/kernel.md). Installing it keeps
`7.3.0-rc6-n1x+` in the GRUB menu. `uname -r` after the reboot is
`7.3.0-rc6-n1x-64m.2+`.

## nvidia-vaapi-driver 0.0.17

File: `nvidia-vaapi-driver_0.0.17-0n1x_arm64.deb`

SHA256: `357a43ee070f1e41ef550b37df5e4b8d161d18b12fca7b66dc57343a27b1a44a`

The image ships `nvidia-vaapi-driver` 0.0.16 as an unpackaged file at
`/usr/lib/aarch64-linux-gnu/dri/nvidia_drv_video.so`. That build allocates
decode memory as local framebuffer memory. On this GPU the framebuffer size
is zero (unified memory, the same class as DGX Spark / GB10), and the
allocation fails. NVDEC stays idle.

Upstream 0.0.17 (tag `v0.0.17`, commit `0ba758d`) sees that zero-framebuffer
GPU and allocates the surfaces from system memory. This package is that
tree, cross-built for arm64 against the Ubuntu 26.10 headers, and it drops
the library in the same path. The revision `0n1x` sorts below a future
Ubuntu `0.0.17-1`, so the archive package can replace it later, and it
sorts above the image's 0.0.16.

Install it on the installed system, then start a new player. No reboot and
no kernel change:

```sh
sudo dpkg -i nvidia-vaapi-driver_0.0.17-0n1x_arm64.deb
```

Checked on this laptop with kernel `7.3.0-rc6-n1x+` and NVIDIA 615.78.08.
`vainfo` reports the direct backend and prints
`Detected unified memory system (IS_ZERO_FB=1)`. With 0.0.16, realising a
1080p surface fails. With this package, a 3840x2160 H.264 file decoded
through VA-API and `nvidia-smi` showed the decoder in use (about 70% for
part of the run, video clock around 2281 MHz).

This package is the decoder half. The browsers still need a build that can
see `libnvidia-rmapi-tegra`. The Firefox snap and the Chromium snap cannot.
Firefox 158 from the Mozilla tarball, with Wayland left on, plays 1440p
smoothly on this laptop once this package is installed. The downloads and
the menu commands are in [docs/video.md](../docs/video.md).
