# Packages to install after Ubuntu is on the NVMe

The ISO already boots kernel `7.3.0-rc6-n1x+`. These packages do not replace
that kernel. `apt upgrade` still will not pull `linux-generic`, because of
the pin in `overlay/etc/apt/preferences.d/n1x-kernel`.

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

Firefox 143 in the menu still draws through XWayland, so a browser frame
is still copied through main memory after it is decoded. That copy is
separate from this package. Players that take VA-API directly (ffmpeg,
and mpv once it is installed) use the decoder without that copy.
