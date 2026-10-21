# Boot

On the ProArt P14, hold the key as soon as you press the power button.

1. Hold **F2** to open the BIOS setup. Set **Secure Boot** to **Disabled**,
   then save and exit. The kernel is unsigned, so this stays disabled after
   the install too.
2. Hold **ESC** to open the firmware boot menu. Choose **EFI USB disk** to
   start the installer from the stick.
3. In GRUB, choose option **3** (desktop on the NVIDIA GPU).

The hardware arguments below are one kernel command line.
`acpi_osi="!Windows 2022"` has to stay literal.

The installed system gets this line from `/etc/default/grub.d/99-n1x.cfg`:

```
console=tty0 acpi=nospcr earlycon mem_sleep_default=s2idle iommu.passthrough=1
power_wrap_drv.usb4_release=0 pci=hpbussize=0x80,hpmmiosize=32M,hpmmioprefsize=32G
log_buf_len=4M
acpi_osi="!Windows 2022"
nvidia_drm.modeset=1 nvidia_drm.fbdev=1
initcall_blacklist=simpledrm_platform_driver_init
```

The live entry adds three timing quirks and leaves snapd running, so the
desktop installer can start:

```
systemd.mask=systemd-udev-settle.service
rd.systemd.device-timeout=5s systemd.device-timeout=5s
systemd.mask=iio-sensor-proxy.service
```

## Why `acpi_osi="!Windows 2022"` is literal

BIOS H7407BA.300 and .303 hide SSPM (`NVDA8400`), COM1-3, SVP0 and HFRP when
`_OSI("Windows 2022")` is true. Linux 6.12 and later answers that string, so
the SSPM mailbox never probes. `power_wrap` then fails and the I2C
controllers, audio DSPs, SoundWire and USB4 host routers stay unbound.
Keyboard, touchpad and speakers depend on that mailbox.

The argument contains a space. It has to be written on the GRUB `linux` line
itself. A GRUB variable that holds the quotes is split, and the quotes are
escaped, so the firmware still sees the Windows 2022 string. Entry 6 omits
the argument on purpose, for comparison. `.303` did not remove the check.

`/proc/cmdline` shows the value as `acpi_osi=!Windows 2022` because of the
space. That form is the one that works: `NVDA8400` status is 15.

## Kernel options that have to be built in

`CONFIG_MTK_POWER_WRAP=y` and `CONFIG_MTK_SSPM_CI=y`. Without them
`i2c-mt65xx` returns `-ENODEV` even when the modules are loaded early.
`IOMMU_DEFAULT_PASSTHROUGH=y` matches `iommu.passthrough=1`. The GPU still
gets its own DMA domain.

The initrd must not carry Bluetooth or wireless modules without the MT7925
firmware. An early `btmtk` probe then loops on a missing firmware file and
resets the USB device. Do not put modprobe snippets in initrd `conf/conf.d/`;
casper sources that directory as shell. The NTFS redirect lives in
`etc/modprobe.d/` inside the initrd, as its own cpio member.

## Order before the desktop

`n1x-nvidia-nodes.service` runs `nvidia-modprobe -c0 -u` once `/dev/dri/card0`
exists, and GDM is ordered after that service. The first GDM start used to
die inside about ten seconds because `/dev/nvidiactl` was missing. The
service returns when the nodes exist. It does not sleep.

`n1x-logdump.timer` fires 20 seconds after boot and only if `/cdrom` exists.
It must not be a oneshot wanted by `multi-user.target`: that ordering put a
25 second sleep and a 120 second sleep on the critical path (2 minutes 55
seconds). `casper-md5check.service` is masked for the same reason.
`geoclue.service` has `TimeoutStartSec=10`.
