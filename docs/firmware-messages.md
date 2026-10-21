# Firmware messages that stay

These lines are still in `dmesg` on a working boot. They were checked
against a device that already works. The image does not try to silence them.

| Message | Why it stays |
| --- | --- |
| `GICv3: Illegal GSI2 translation request` | Firmware, twice at boot. Input and PCI still enumerate. |
| `arch_timer: VHE-capable CPU without EL2 virtual timer interrupt` | Firmware. The timer works. |
| `mtk_pwrap NVDA6210:00: Not find device config data` | The wrapper still probes. All seven I2C controllers bind. |
| `mpam: No MSC devices found in firmware` | This SoC has no MPAM MSC. |
| `i2c_hid_acpi ... incorrect report (255 vs 11/7/14)` | PixArt feature report the pad does not serve. The libinput quirk is what makes the cursor move. |
| `hid-multitouch 222A:5553: failed to fetch feature` | Same class. The touchscreen works. |
| `asus 0B05:4B42: Asus handshake a returned invalid response` | hid-asus still binds the keyboard. |
| `usb usb2-port1: Failed to created device link ... NVDA8100` | USB4 device link during probe. USB ethernet on that bus works. |
| `mtk_sdw_mc ... -517` | `-EPROBE_DEFER` until the CS42L43 and CS35L56 slaves attach. |
| `driver name too long 'mt8901-soundwire' -> 'mt8901-soundwir'` | The kernel truncates the card name. UCM uses the truncated name. |
| `DL0_FE: no backend DAIs enabled` | Printed once, before PipeWire loads UCM. |
| `supply VDD_* not found, using dummy regulator` | The DSDT does not describe the Cirrus rails. The codecs still probe. |
| `gpio: Failed to set debounce-timeout: -22` | One debounce request is rejected. Keys and the pad work. |
| `arm-smmu-v3: 2-level strtab` / `no priq irq` | SMMU geometry. `iommu.passthrough=1` is set. |
| `thunderbolt-platform NVDA8100: RX overflow` | A few events at probe. |
| `HCI Enhanced Setup Synchronous Connection ... not supported` | The chip advertises a command it does not implement. hci0 registers. |
| `DMA mask not set` | HID sensor and the Cirrus platform devices. Drivers continue. |
| `nvidia: loading out-of-tree module taints kernel` | The open module is unsigned. |
| `koobInitAperture_HAL ... NV_ERR_NOT_SUPPORTED` | Once, at probe. `nvidia-smi` still sees the RTX Spark N1X. |
| `GPT: Alternate GPT header not at the end of the disk` | Hybrid ISO. Expected on the stick, not on an installed disk. |
| `overlayfs: xino=off` | The live squashfs has no file handles. |
| `bpf-restrict-fs: Failed to load BPF object` | systemd LSM setup on the live overlay. |

`systemd-oomd` starts at boot and notes that there is no swap. That line is
the service opening its socket. It is not a process being killed.

The old `ntfs` driver and `/dev/sda3: Can't open blockdev` are not in this
table. The first is redirected to `ntfs3` in the initrd. The second was the
live log dump opening a partition of the disk that is already mounted as the
ISO. The dump no longer does that.
