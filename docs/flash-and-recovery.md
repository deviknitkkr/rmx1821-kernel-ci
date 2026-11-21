# Flash and recovery

**The 69 Hz image has been tested and bootlooped; the user rolled back.** The phone is back
on the stock kernel, and `boot` hashes to `b5c65e59…` (the backup). A **stock-timing
control** (run `44932757133`, no vfp/PLL change) is building to separate the panel timing
from the clang-18 toolchain. The marker test below is still the zero-risk way to learn
whether `fastboot boot` works on this LK.

## Step 0: prove `fastboot boot` loads the uploaded image

Download on the host:

```
https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/download/ramboot-diag-v1/boot-ramtest-marker.img
sha256 026cb9a6b590990da7806058c9a4a992bb91f265bd214f195d0b4d7b9d3ca55e
```

```bash
fastboot boot boot-ramtest-marker.img
# after Android is up:
cat /proc/cmdline | tr ' ' '\n' | grep ramtest
```

- `ramtest=stock_marker` → the upload is honored. Continue below.
- no output → LK ignores uploaded images; only a real flash can test the kernel
  (`fastboot flash boot`, with `backup-v1` as the rollback).

This image has the **stock kernel** (sha `98e4a881…`), so it is risk-free. If it boots,
`uname -v` will still show 2021 — that is expected; the marker is the test.

## Test first: RAM boot (writes nothing)

> **Do not boot the 69 Hz image again** — it bootlooped. Use this procedure for the
> stock-timing control or a conservative variant once it is built.

Use the **tagged** 69 Hz image so the boot itself proves which kernel ran:

```
https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/download/ramboot-diag-v1/boot-69hz-tagged.img
sha256 4e6871ceac7bd82af7ec0bbe81bb0ed116a1cd3a7a8cd297d3965bb429d9a360
```

(The original `boot-69hz-v1/boot-69hz-magisk.img`, sha256 `7c5975d4…4d568bd`, has the same
kernel without the cmdline marker.)

Download it on the **host** (Pixel/PC), then:

```bash
# phone: su -c 'reboot bootloader'   (or Vol Up + Vol Down + Power)
fastboot boot boot-69hz-tagged.img
# after boot:
uname -v                                          # expect #1 ... Sat Nov 21 2026
cat /proc/cmdline | tr ' ' '\n' | grep ramtest    # expect ramtest=69hz_marker
```

- nothing is written; a normal reboot returns to the current kernel
- Magisk comes up from the image's ramdisk, so root works
- if the panel refuses the timing, power-cycle; the boot partition is untouched

Checks once it boots:

```bash
# Developer options -> Show refresh rate  (expect ~69)
dumpsys display | grep supportedModes
dumpsys SurfaceFlinger | grep -E "VSYNC period|primary range"
su -c 'dmesg | grep -iE "disagrees about version|unknown symbol"'
```

Wi-Fi working is the quick proof that vendor modules still load.

## Backups (public, no auth)

Release: `https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/tag/backup-v1`

| Asset | Size | sha256 |
|---|---|---|
| `boot_current_magisk_RMX1821.img` | 33,554,432 B | `b5c65e5975f27b44bdfdccbf999b35b503d570b14cc014bb0962f102ba7b800c` |
| `dtbo_stock_RMX1821.img` | 16,777,216 B | `1f180528f0167ec590cbb644490b19103db146a08d7f082cb60dac749fcbfe7f` |
| `SHA256SUMS` | — | verify with `sha256sum -c SHA256SUMS` |
| `RECOVERY.txt` | — | short version of this document |

`boot_current_magisk_RMX1821.img` is the **current working boot**, not factory-stock. Its
ramdisk contains Magisk's `.backup` and `overlay.d`; `magiskboot cpio ramdisk.cpio "exists
.backup"` exits 0 (it prints nothing on v31.1).

A factory-stock boot would have to come from the LineageOS 18.1 ZIP
(`18.1-20210621-UNOFFICIAL-RMX1821`). Flashing that would remove Magisk but still not touch
`/data`.

## Prerequisites on the phone

- root (`su`), Magisk installed
- `magiskboot` at `/data/adb/magisk/magiskboot`
- battery above ~50% (a power loss mid-write corrupts `boot`)
- the artifacts downloaded (see below)
- optionally the base boot image for a local repack

## 1. Download the artifacts

The workflow now produces a **flashable image already**:

```bash
gh run list --repo deviknitkkr/rmx1821-kernel-ci --workflow build.yml --limit 5
gh run download <run-id> --repo deviknitkkr/rmx1821-kernel-ci \
  -n boot-69hz-magisk -D ~/boot-out
ls -l ~/boot-out
sha256sum -c ~/boot-out/SHA256SUMS
```

The kernel-only artifact is still available as `-n Image-69hz`
(`Image.gz` + `.config` + `report.txt`).

## 2. Get a flashable boot image

### Option A — use the prebuilt image (default)

`boot-69hz-magisk.img` from release `boot-69hz-v1`, or `boot.img` from the CI artifact
`boot-69hz-magisk`, already has the new kernel inside the Magisk-patched boot. The CI/local
repack verified `.backup`, the kernel contents and the 32 MiB limit.

One extra check: confirm the base it was built from is still the boot partition the phone
runs. The base is `boot_current_magisk_RMX1821.img`, sha256
`b5c65e59…b800c` (the 69 Hz image itself is `7c5975d4…4d568bd`):

```bash
su -c 'dd if=/dev/block/by-name/boot bs=1M 2>/dev/null' | sha256sum
```

If it matches, skip to step 3. If it **differs** (Magisk was re-patched, an OTA landed,
etc.), use Option B so the ramdisk is the live one.

### Option B — repack locally from the live boot

```bash
su -c '
  rm -rf /data/local/tmp/repack && mkdir -p /data/local/tmp/repack &&
  cd /data/local/tmp/repack &&
  cp /data/data/com.termux/files/home/rmx1821-backup/boot_stock.img live-boot.img &&
  /data/adb/magisk/magiskboot unpack live-boot.img &&
  cp /data/data/com.termux/files/home/kernel-out/Image.gz kernel &&
  /data/adb/magisk/magiskboot repack live-boot.img new-boot.img
'
```

`boot_stock.img` is the local dump and is byte-identical to the release asset
`boot_current_magisk_RMX1821.img`; either works. If you downloaded the release asset, use
its path in place of `boot_stock.img`.

`magiskboot repack` consumes `kernel`, `kernel_dtb`, `ramdisk.cpio` and `vbmeta` from the
current directory. Only `kernel` was replaced; the Magisk ramdisk is untouched.

## 3. Verify before flashing (not optional)

Even with Option A, re-run this on the phone:

```bash
su -c '
  rm -rf /data/local/tmp/verify && mkdir -p /data/local/tmp/verify &&
  cd /data/local/tmp/verify &&
  cp /data/local/tmp/repack/new-boot.img . &&
  /data/adb/magisk/magiskboot unpack new-boot.img &&
  /data/adb/magisk/magiskboot cpio ramdisk.cpio "exists .backup" &&
  stat -c "size=%s" new-boot.img
'
```

(With Option A, copy `~/boot-out/boot.img` to `/data/local/tmp/repack/new-boot.img` first.)

Expected:

- `KERNEL_FMT [gzip]`, `RAMDISK_FMT [gzip]`, `HEADER_VER [1]`
- `KERNEL_DTB_SZ [138882]` (unchanged)
- `exists .backup` exits 0 (magiskboot v31.1 prints nothing; non-zero = missing)
- `size` ≤ `33554432`

Also confirm the kernel itself came from the right build:

```bash
# on the CI log, or from the build output:
#   cat out/include/config/kernel.release   ->  4.14.235-M.A.P
```

## 4. Flash

> **Do not flash until the Step 0 marker test passes and the 69 Hz image has booted from
> RAM.** If RAM boot is unproven, flashing is the only way to test the kernel — do it only
> with the Pixel/laptop and `boot_current_magisk_RMX1821.img` ready to restore.

```bash
su -c 'dd if=/data/local/tmp/repack/new-boot.img of=/dev/block/by-name/boot bs=1M conv=fsync'
su -c 'sync'
su -c 'reboot'
```

`boot` is not mounted while Android runs, so writing it live is safe and no recovery boot is
needed.

## 5. Verify the result

- Developer options → **Show refresh rate** → should read ~69 (variant 69) or ~72 (variant 72).
- `dumpsys display | grep supportedModes` → should report the new `fps=` if HWC follows the
  kernel.
- Compare scroll smoothness in a long list.

If the overlay still reads 60 but the panel is being driven faster, see "HWC mismatch" below.

## Recovery

### Fastboot (the normal path)

1. Power off.
2. Hold **Vol Up + Vol Down + Power** until the fastboot screen appears.
   (Alternatives when the system still boots: `adb reboot bootloader`, or
   `su -c 'reboot bootloader'`.)
3. Connect USB to the **Pixel or a laptop** with platform-tools.
4. `fastboot devices`
5. `fastboot flash boot boot_current_magisk_RMX1821.img`
6. `fastboot reboot`

Only flash `dtbo` if the device tree was changed:

```bash
fastboot flash dtbo dtbo_stock_RMX1821.img
```

Restoring `boot` does **not** touch `/data`. Encryption keys live in `metadata` + `userdata`,
which are never written.

### Flashing from the Pixel (USB-OTG)

The Pixel has been used as the fastboot host for this device, so it is the preferred rescue
if the phone won't boot. Use a USB-C to micro-USB cable/adapter and platform-tools.

If `fastboot devices` shows nothing, the Pixel's Termux may lack raw USB access (unrooted
apps cannot open the USB device); in that case use a laptop. The backup release is public,
so no GitHub auth is needed to download the restore image.

### Last resort

MTK BROM / SP Flash Tool with a full firmware package. This **wipes userdata**. Only if
fastboot is unreachable and the device is truly dead.

## What each failure looks like

| Symptom | Meaning | Action |
|---|---|---|
| Bootloop after installing the 69 Hz image | kernel/panel hang before Android; rollback works | restore boot (done); test the stock-timing control, then conservative vfp |
| `fastboot boot` prints OKAY but the stock kernel is running after reboot | LK ignored/crashed on the uploaded image (`bootreason=lk_crash`); nothing was written | run the marker test; if no marker, only a real flash can test the kernel |
| Black screen / no backlight after logo | panel rejected the new timing | restore boot |
| Boot logo, then reboot loop | kernel panic | restore boot |
| Boots, but scrolling is *worse* / beat-y | HWC still reports 60 while panel runs faster | restore boot, see `status.md` |
| Boots, but Wi-Fi/touch/etc. dead | vendor module vermagic mismatch (wrong kernel release string) | restore boot; rebuild with the exact defconfig |
| Boots, overlay shows 69/72, smooth | success | keep, then try variant 72 |

## Never flash

`userdata`, `metadata`, `persist`, `nvram`, `nvcfg`, `nvdata`, `frp`, `seccfg`, `proinfo`,
`otp`, `protect1/2`, `md1img`, `lk`, `lk2`, `spmfw`, `scp*`, `sspm*`, `tee*`, `gz*`.

Those hold IMEI, calibration, DRM keys, and the bootloader. Losing them is not recoverable
with a boot image.

## If the phone is lost or stolen

Revoke the GitHub token used by this project:
`https://github.com/settings/tokens` — scopes `repo`, `workflow`, `read:org`.

It lives on the phone at `~/.cache/gh_token` and `~/.config/gh/hosts.yml`.
