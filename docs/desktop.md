# Desktop, GPU and decode

Userspace and the kernel module are both NVIDIA 615.78.08. Do not install
Ubuntu `libnvidia-gl-615` (615.71.09) over these libraries.

## GBM and EGL

`libgbm` loads `/usr/lib/aarch64-linux-gnu/gbm/nvidia-drm_gbm.so`. That name
is a symlink to `libnvidia-allocator.so.1`, which exports `gbmint_get_backend`.
`libnvidia-egl-gbm.so.1` is only the EGL external platform
(`/usr/share/egl/egl_external_platform.d/15_nvidia_gbm.json`). It is not a
GBM backend.

These sonames have to exist, and `ldconfig` has to see them:

- `libEGL_nvidia.so.0` -> `libEGL_nvidia.so.615.78.08`
- `libGLX_nvidia.so.0` -> `libGLX_nvidia.so.615.78.08`
- `libnvidia-allocator.so.1` -> `libnvidia-allocator.so.615.78.08`
- `libnvidia-cfg.so.1` -> `libnvidia-cfg.so.615.78.08`

`libEGL_nvidia` also needs `libnvidia-rmapi-tegra.so.615.78.08`, or
`eglInitialize` fails with no message.

`/dev/dri/renderD128` is mode 660, group `render`. The desktop user has to
be in that group. Without it, NVIDIA hides `EGL_DRM_DEVICE_FILE_EXT` and the
GBM platform returns no display. A session that started before the libraries
or the group membership keeps llvmpipe until the next login.

Checked result: `gbm` plus `eglGetPlatformDisplay(EGL_PLATFORM_GBM_KHR)`
reports `vendor=NVIDIA version=1.5`. `nvidia-smi` lists `gnome-shell` and
`Xwayland` as graphics clients. The journal says the device was added with
atomic modesetting and that a high priority EGL context was obtained.

`/usr/bin/nvidia-modprobe` (setuid) and
`/usr/lib/udev/rules.d/71-nvidia.rules` create `/dev/nvidiactl`,
`/dev/nvidia0` and `/dev/nvidia-uvm` when `nvidia-drm` loads.
`n1x-nvidia-nodes.service` runs the same command before GDM.

## VA-API and Firefox

`LIBVA_DRIVER_NAME=nvidia`. The image file
`dri/nvidia_drv_video.so` is `nvidia-vaapi-driver` 0.0.16, together with
`libnvcuvid.so.1` and `vdpau/libvdpau_nvidia.so.1` from the 615.78.08 tree.
`vainfo` lists AV1, HEVC, H.264 and VP9 on the direct backend. That list is
only the capability probe. 0.0.16 then allocates each surface as local
framebuffer memory. This GPU reports a zero framebuffer (unified memory),
the allocation is rejected, and NVDEC stays at 0%.

Upstream 0.0.17 detects `IS_ZERO_FB` and allocates those surfaces from
system memory. The arm64 package is
[debs/nvidia-vaapi-driver_0.0.17-0n1x_arm64.deb](../debs/nvidia-vaapi-driver_0.0.17-0n1x_arm64.deb).
Install it with `dpkg -i` after Ubuntu is on the NVMe. The running kernel
stays `7.3.0-rc6-n1x+`. On this laptop, 0.0.16 logged
`Unable to realise surface` for a 1920x1088 frame. The same file with
0.0.17 decoded, and a 3840x2160 H.264 stream drove the NVDEC clock
(decoder utilisation about 70% during the run).

Playback in Firefox is still a separate limit. Firefox 143 shows each
decoded frame through XWayland, so the picture is copied through main
memory. Chromium on Wayland is smoother and still hitches, because the
snap does not receive `libnvidia-rmapi-tegra` and composites on the CPU.

Firefox 143 is the Mozilla aarch64 tarball in `/opt/firefox`. The menu
entry is `/usr/local/share/applications/firefox_firefox.desktop` and starts
that binary. The Ubuntu Firefox snap is installed, and it is not the menu
entry. Snap confinement leaves out `libnvidia-rmapi-tegra`, so the snap
draws with Mesa and does not show up in `nvidia-smi`. Policies in
`/opt/firefox/distribution/policies.json` lock
`media.hardware-video-decoding.force-enabled`. The preference
`media.ffmpeg.vaapi.enabled` does not exist in this Firefox.
`MOZ_ENABLE_WAYLAND=0` is set on that desktop entry only, because this
build crashes on mutter 51's `wp_image_description_v1`.
`LIBVA_DRIVER_NAME=nvidia` and `MOZ_DISABLE_RDD_SANDBOX=1` are in
`/etc/environment`. The second one lets the decoder open the render node.

The tarball dlopens `libavcodec.so.61` at most. Ubuntu 26.10 has
`libavcodec62`, so decode stays on the bundled ffvpx plus VA-API. Firefox
gives its `vaapitest` helper two seconds. A manual run prints
`VAAPI_SUPPORTED`. Under `strace` the helper exceeds that limit.

## Touchpad

PixArt `093A:4F02`. libinput sees pressure 0..65535 while the real values
are tiny, so the default thresholds drop every contact.
`/etc/libinput/local-overrides.quirks` sets `AttrPressureRange=none` and
`AttrPalmPressureThreshold=0` for DMI `ProArtP14H7407BA`. GNOME applies that
only to a device it opens after the file exists.

## Speakers

The card name is truncated to `mt8901-soundwir`. UCM lives in
`usr/share/alsa/ucm2/MediaTek/mt8901-soundwir/` and the `conf.d` symlink.
The profile is `HiFi-cs42l43-cs35l56x2.conf` (CS42L43 and two CS35L56 on
SoundWire). Cirrus tuning `cs35l56-b0-dsp1-misc-104333a1-spkid0/1` is in the
image. `mtk_sdw_mc` returns `-517` until those slaves attach; the card comes
up after that.
