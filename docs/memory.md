# Memory

The machine has 128 GB. BIOS H7407BA.303 reserves about 62.5 GiB, starting
just after 8 GiB, as the Windows GPU heap (`1fd000000-119fffffff` in
`/proc/iomem`). The open NVIDIA driver does not use that heap. CUDA allocates
from the one coherent pool, so the reservation sits idle and `free` shows
about 61 GiB.

This is not automatic, and it is not a separate GPU heap you size at
runtime. DGX Spark firmware does not carve the Windows heap out, which is
why that machine shows the full 128 GB. On this BIOS the carveout is an EFI
variable.

## Firmware setting

`EXPOSED_DISP_FRM` (`4b086294-0740-4f71-a7b5-85a329c7ec5f`) is the variable
NVIDIA and the OEMs agreed on. The first four data bytes are the reservation
in MiB. Shipped value `0xFA00` is 62.5 GiB. `0x1000` is 4 GiB. The last byte
asks the firmware to apply the size. The layout between those fields has to
match what the firmware shipped; any other layout is left alone.

`/usr/local/sbin/n1x-gpu-memory` writes 4 GiB. efivarfs rejects a shell
redirect (`O_TRUNC`, and the write must be one `write()`). The script writes
through Python. The new size takes effect at the next boot.
`n1x-gpu-memory.service` runs it once per boot and does nothing when the
value is already 4 GiB. The variable is in NVRAM. Wiping the disk,
reinstalling Ubuntu, or restoring Windows leaves the 4 GiB value in place.

On the 2026-10-20 boot after that write, `MemTotal` was 120.3 GiB. The early
kernel line was `Memory: 125505320K/133845504K available` with about 7.7 GiB
still reserved (the 4 GiB firmware reservation plus the usual holes).
The working maximum used on this laptop is 112 GB. 96 GB of that was tested.

The older kernel parameter `efi_reclaim_reserved=58G@8G` needs Omarchy patch
1050, which is not in `7.3.0-rc6-n1x`. Omarchy has dropped that parameter in
favour of the EFI variable. Putting the parameter on the command line of
this kernel does nothing.

## CUDA

`nvidia-smi` reports `Memory-Usage: Not Supported`. There is no discrete
VRAM pool. A process allocates from the 120 GiB.

The allocation has to use the 64-bit driver entry `cuMemAlloc_v2`. The
unversioned `cuMemAlloc` takes a 32-bit size. A 16 GiB request then becomes
an invalid argument, and a 90 GiB request is truncated. Writing past the
truncated mapping raises Xid 31 (`FAULT_PDE`) and later `cuInit` calls fail
until the next boot. `nvidia-smi --gpu-reset` refuses because this GPU is
the primary display.

Checked on this machine with `cuMemAlloc_v2` and `cuMemsetD32_v2`, pattern
`0xA11C0DE0` at both ends of the buffer:

- 16 GiB held, and `MemAvailable` dropped by about 16 GiB.
- 90 GiB held, and `MemAvailable` dropped from about 118 GiB to about 27 GiB,
  then returned after the free.

Each `cuMemsetD32_v2` call stays inside a 4 GiB span so the word count fits
in 32 bits.
