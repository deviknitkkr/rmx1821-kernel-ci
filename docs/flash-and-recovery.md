# Flash and recovery

**Nothing in this document has been executed on the phone yet.** Read the whole thing before
the first flash.

## Backups (public, no auth)

Release: `https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/tag/backup-v1`

| Asset | Size | sha256 |
|---|---|---|
| `boot_current_magisk_RMX1821.img` | 33,554,432 B | `b5c65e5975f27b44bdfdccbf999b35b503d570b14cc014bb0962f102ba7b800c` |
| `dtbo_stock_RMX1821.img` | 16,777,216 B | `1f180528f0167ec590cbb644490b19103db146a08d7f082cb60dac749fcbfe7f` |
| `SHA256SUMS` | — | verify with `sha256sum -c SHA256SUMS` |
| `RECOVERY.txt` | — | short version of this document |

`boot_current_magisk_RMX1821.img` is the **current working boot**, not factory-stock. Its
ramdisk contains Magisk's `.backup` and `overlay.d`, verified with
`magiskboot cpio ramdisk.cpio "exists .backup"` → true.

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

### Option A — use the CI-repacked image (default)

`boot-out/boot.img` already has the new kernel inside the Magisk-patched boot, and the CI
verified `.backup`, the kernel contents and the 32 MiB limit.

One extra check: confirm the base it was built from is still the boot partition the phone
runs. The release asset's hash is
`b5c65e5975f27b44bdfdccbf999b35b503d570b14cc014bb0962f102ba7b800c`:

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
  cp /data/data/com.termux/files/home/rmx1821-backup/boot_current_magisk_RMX1821.img . &&
  /data/adb/magisk/magiskboot unpack boot_current_magisk_RMX1821.img &&
  cp /data/data/com.termux/files/home/kernel-out/Image.gz kernel &&
  /data/adb/magisk/magiskboot repack boot_current_magisk_RMX1821.img new-boot.img
'
```

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
- `exists .backup` → `true` (Magisk preserved)
- `size` ≤ `33554432`

Also confirm the kernel itself came from the right build:

```bash
# on the CI log, or from the build output:
#   cat out/include/config/kernel.release   ->  4.14.235-M.A.P
```

## 4. Flash

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
3. Connect USB to a computer with platform-tools.
4. `fastboot devices`
5. `fastboot flash boot boot_current_magisk_RMX1821.img`
6. `fastboot reboot`

Only flash `dtbo` if the device tree was changed:

```bash
fastboot flash dtbo dtbo_stock_RMX1821.img
```

Restoring `boot` does **not** touch `/data`. Encryption keys live in `metadata` + `userdata`,
which are never written.

### Flashing from another phone (Pixel, USB-OTG)

Hardware works (USB-C host + C-to-micro-USB cable), but `fastboot` in Termux on an
**unrooted** phone needs raw USB access that Android does not grant to apps. `termux-usb` can
sometimes pass a device fd to a program, but `fastboot` does not consume one. Treat this as
unreliable; a laptop is the dependable rescue.

### Last resort

MTK BROM / SP Flash Tool with a full firmware package. This **wipes userdata**. Only if
fastboot is unreachable and the device is truly dead.

## What each failure looks like

| Symptom | Meaning | Action |
|---|---|---|
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
