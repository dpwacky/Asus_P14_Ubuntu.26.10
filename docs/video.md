# Video acceleration

The GNOME desktop is already a native NVIDIA client. Browser video is a
separate piece, and the Ubuntu snap store does not pick it up. The three
steps below are userspace. They do not replace the kernel. A 4K or 8K
frame also needs a kernel that can allocate one block larger than 4 MiB.
That package is [kernel.md](kernel.md), and it is not in the ISO.

An older paragraph in this handbook said that 1080p is only just usable
and that 4K stutters. That was Firefox 143 on the image kernel, drawing
each frame through XWayland, with `nvidia-vaapi-driver` 0.0.16. The ISO
still ships that combination. The measurement below is Firefox 158 after
the three steps and the 64 MiB kernel.

## What was measured

Firefox 158.0.1, Wayland, `LIBVA_DRIVER_NAME=nvidia`,
`MOZ_DISABLE_RDD_SANDBOX=1`, kernel `7.3.0-rc6-n1x-64m.2+`, on the internal
2880x1800 panel. Each file is an 8 second, 30 fps H.264 test pattern made
with libx264. The pattern compresses easily, so the bitrates are about
1.5, 2.4 and 7.5 Mbit/s. This is not a YouTube 8K AV1 stream.

The RDD log says `VA-API FFmpeg init successful` and
`IsHardwareAccelerated=true`. Each decoded frame is logged as a VA-API
frame. The first frame is late once. After that the decode time stays
under 1 ms, against a 33 ms frame. `dmesg` had no `NV_ERR_NO_MEMORY`.
`nvidia-smi` decoder utilisation stayed low because the frames finish
quickly, not because decode fell back to the CPU.

| File | Dropped / presented, at 9 s | Playhead | First frame | Later frames, average |
| --- | --- | --- | --- | --- |
| 1920x1080 | 13 / 233 | 7.75 s of 8 s | 104 ms | 0.49 ms |
| 3840x2160 | 9 / 231 | 7.64 s of 8 s | 184 ms | 0.45 ms |
| 7680x4320 | 6 / 229 | 7.61 s of 8 s | 247 ms | 0.53 ms |

The whole Firefox process tree still uses CPU. That is the browser and
the compositor. The decoder log is the VA-API path above, at about half a
millisecond a frame.

## Why a snap stays on the CPU

NVIDIA EGL on this machine needs
`libnvidia-rmapi-tegra.so.615.78.08` from
`/usr/lib/aarch64-linux-gnu`. A normal program on the host loads that file
and shows up in `nvidia-smi`.

Firefox and Chromium from the Ubuntu store are snaps. At start,
`snap-confine` hides `/var/lib/snapd/lib/gl` under a private directory and
copies in an allowlist of NVIDIA libraries (`libEGL_nvidia`,
`libnvidia-eglcore`, `libnvidia-glcore`, and the usual desktop names).
`libnvidia-rmapi-tegra` is not on that list. Inside the snap, `ldd` on
`libEGL_nvidia.so.615.78.08` reports
`libnvidia-rmapi-tegra.so.615.78.08 => not found`, even though the file
sits on the host and also in the on-disk `/var/lib/snapd/lib/gl`. The snap
then builds the page on the CPU and hands the window to GNOME.

Installing "Firefox" or "Chromium" from the store, or with `snap install`,
does not change that. The copies that work are a Mozilla tarball and a
Google Chrome `.deb`. Both run outside the snap and use the host libraries.

The image's own menu Firefox is already a tarball, at `/opt/firefox`,
version 143.0.4. That build crashes on this GNOME's Wayland color manager
(`wp_image_description_v1`), so the menu starts it with
`MOZ_ENABLE_WAYLAND=0`. The window is then drawn through XWayland: decode,
copy through main memory, upload again. Version 158 speaks the current
Wayland protocol and can sit on the NVIDIA GPU directly.

## 1. Replace the VA-API driver

The image file `/usr/lib/aarch64-linux-gnu/dri/nvidia_drv_video.so` is
`nvidia-vaapi-driver` 0.0.16. It asks for local framebuffer memory. This
GPU has none (unified memory, the same class as DGX Spark / GB10), so every
surface fails and NVDEC stays at 0%. `vainfo` can still list H.264, HEVC,
VP9 and AV1. That list is only the probe.

Upstream 0.0.17 (tag `v0.0.17`, commit `0ba758d`) sees `IS_ZERO_FB` and
allocates the surfaces from system memory. The arm64 package built for this
image is in the repository:

[debs/nvidia-vaapi-driver_0.0.17-0n1x_arm64.deb](../debs/nvidia-vaapi-driver_0.0.17-0n1x_arm64.deb)

SHA256 `357a43ee070f1e41ef550b37df5e4b8d161d18b12fca7b66dc57343a27b1a44a`.

On the installed system:

```sh
sudo dpkg -i nvidia-vaapi-driver_0.0.17-0n1x_arm64.deb
```

No reboot. `uname -r` stays `7.3.0-rc6-n1x+`. Checked here: 0.0.16 logged
`Unable to realise surface` for a 1920x1088 frame. The same file with this
package decoded, and a 3840x2160 H.264 stream put the decoder around 70%
with the video clock near 2281 MHz.

`/etc/environment` already contains `LIBVA_DRIVER_NAME=nvidia` and
`MOZ_DISABLE_RDD_SANDBOX=1`. Leave both. The second one lets Firefox open
`/dev/dri/renderD128`.

## 2. Firefox 158 from the Mozilla tarball

This is the browser in the measurement above. The download
is the official aarch64 build, not the snap and not an `apt` package.
Ubuntu 26.10's `firefox` package is only a wrapper that installs the snap.

```sh
sudo apt-get install -y curl
curl -fL -o /tmp/firefox-158.0.1.tar.xz \
  https://download-installer.cdn.mozilla.net/pub/firefox/releases/158.0.1/linux-aarch64/en-US/firefox-158.0.1.tar.xz
mkdir -p /tmp/firefox-unpack
tar -C /tmp/firefox-unpack -xJf /tmp/firefox-158.0.1.tar.xz
sudo rm -rf /opt/firefox-158
sudo mv /tmp/firefox-unpack/firefox /opt/firefox-158
sudo mkdir -p /opt/firefox-158/distribution
sudo cp -a /opt/firefox/distribution/policies.json /opt/firefox-158/distribution/policies.json
```

The policies file locks hardware video decoding on. `/opt/firefox` (143)
stays where it is, so the old binary is still there.

Point every menu line that still runs `/opt/firefox/firefox` at the new
binary, and drop `MOZ_ENABLE_WAYLAND=0`. On a fresh install the menu file
is `/usr/local/share/applications/firefox_firefox.desktop`. If a copy
exists in `~/.local/share/applications/`, that copy wins, so change that
one too. Each `Exec=` line should look like this:

```
Exec=env MOZ_ENABLE_WAYLAND=1 /opt/firefox-158/firefox %u
```

The same prefix belongs on the `--new-window`, `--private-window` and
`--ProfileManager` lines. Quit any Firefox that is already open. A process
started from 143 keeps the XWayland path until you close it. Then start
Firefox from the menu. `nvidia-smi` lists `/opt/firefox-158/firefox` as a
graphics client. The frame counts are in the measurement at the top of
this page.

## 3. Chrome from a .deb, with an explicit GPU launch

The Chromium snap has the same missing library as the Firefox snap. Google
publishes an arm64 `.deb` that links the host EGL stack. The build used
here is 156.0.8078.47.

```sh
curl -fL -o /tmp/google-chrome-stable_current_arm64.deb \
  https://dl.google.com/linux/direct/google-chrome-stable_current_arm64.deb
sudo dpkg -i /tmp/google-chrome-stable_current_arm64.deb
```

This Chrome rejects Vulkan on its Wayland backend, and without an extra
switch it logs `Should skip nVidia device named: nvidia-drm` and ignores
the render node. The launcher below turns Vulkan off, selects ANGLE
OpenGL, and turns the NVIDIA VA-API path on. Save it as
`/usr/local/bin/chrome-n1x` and `chmod 755` it:

```sh
#!/bin/sh
export LIBVA_DRIVER_NAME=nvidia
exec /opt/google/chrome/chrome \
  --ozone-platform=wayland \
  --use-angle=gl \
  --disable-features=Vulkan \
  --enable-features=VaapiOnNvidiaGPUs,VaapiVideoDecodeLinuxGL,VaapiIgnoreDriverChecks \
  --ignore-gpu-blocklist \
  "$@"
```

Copy `/usr/share/applications/google-chrome.desktop` to
`/usr/local/share/applications/google-chrome.desktop` and replace each
`Exec=/usr/bin/google-chrome-stable` with `Exec=/usr/local/bin/chrome-n1x`.
The menu item is then Google Chrome, not the Chromium snap. With that
launcher the GPU process shows up in `nvidia-smi`.

## What the image itself still does

The ISO is unchanged. It still ships driver 0.0.16, Firefox 143 started
with `MOZ_ENABLE_WAYLAND=0`, and kernel `7.3.0-rc6-n1x+`. Version 143
crashes as a Wayland client on this desktop, which is why the image menu
keeps XWayland for that binary. The steps above are what you run after the
installer has finished. A later Ubuntu package of `nvidia-vaapi-driver`
0.0.17 or newer can replace `0.0.17-0n1x`. Leave the NVIDIA 615.78.08
libraries alone.

The image kernel's largest physically contiguous allocation is 4 MiB. A 4K
luma plane is about 8 MiB, so 0.0.17 can decode and still log
`NV_ERR_NO_MEMORY` for that plane. After the ISO install, install
`linux-image-7.3.0-rc6-n1x-64m.2` from
[ponne.nu](https://www.ponne.nu/linux-image-7.3.0-rc6-n1x-64m.2_7.3.0-rc6.64m.3_arm64.deb)
and reboot. `uname -r` is then `7.3.0-rc6-n1x-64m.2+`. That kernel raises
the limit to 64 MiB. On this laptop, with that limit, one second of
3840x2160 H.264 decoded through VA-API with no `NV_ERR_NO_MEMORY` in
`dmesg`. The Firefox frame counts for 1080p, 4K and 8K are at the top of
this page.
The download and the checksum are in [kernel.md](kernel.md).
