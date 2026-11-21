# RMX1821 kernel — raised panel refresh

Builds `Image.gz` for the Realme 3 (`RMX1821`, MT6771, ILI9881 INX HD+ panel) with the
panel's vertical front porch reduced, optionally with a 4.4% DSI PLL bump.

| variant | vfp | PLL | expected refresh |
|---|---|---|---|
| `69` | 236 -> 16 | 360 (unchanged) | ~69.2 Hz |
| `72` | 236 -> 16 | 360 -> 376 | ~72.3 Hz |

`69` does not change the DSI link rate at all — it only removes frame blanking.

## Full documentation

Everything needed to understand, continue, or recover this project lives in
**[`docs/`](docs/README.md)**:

| File | Contents |
|---|---|
| [`docs/README.md`](docs/README.md) | handoff index, TL;DR, AI instructions, repo map |
| [`docs/device-and-display.md`](docs/device-and-display.md) | hardware, probe outputs, panel research, the math |
| [`docs/build.md`](docs/build.md) | CI pipeline, the three build bugs and their fixes, local build |
| [`docs/flash-and-recovery.md`](docs/flash-and-recovery.md) | repack, verify, flash, full recovery procedure |
| [`docs/status.md`](docs/status.md) | timeline, run IDs, verified vs. assumed, next steps |

Start there if this device is lost, unbootable, or you are picking the project up cold.

## Safety properties

- `CONFIG_LOCALVERSION="-M.A.P"` is preserved, so the built kernel's release string is
  `4.14.235-M.A.P` and vendor modules still pass vermagic checks.
- `CONFIG_MODVERSIONS=y` CRCs are generated from source by genksyms, not by the compiler,
  so module ABI is unchanged.
- Only `vertical_frontporch` (and optionally `PLL_CLOCK`) change; no headers, no exported
  symbols, no module rebuilds.

## After the workflow finishes

Download the artifact on the phone:

```bash
gh run list --repo deviknitkkr/rmx1821-kernel-ci --workflow build.yml --limit 3
gh run download <run-id> --repo deviknitkkr/rmx1821-kernel-ci -n Image-69hz -D ~/kernel-out
```

Validate, repack **from the current (Magisk-patched) boot**, and flash — full detail in
[`docs/flash-and-recovery.md`](docs/flash-and-recovery.md):

```bash
cd /data/local/tmp/repack
cp ~/kernel-out/Image.gz kernel
/data/adb/magisk/magiskboot repack boot_current_magisk_RMX1821.img new-boot.img

# sanity: still Magisk-patched, kernel gzip, size fits the 32 MiB partition
/data/adb/magisk/magiskboot unpack new-boot.img
/data/adb/magisk/magiskboot cpio ramdisk.cpio "exists .backup"
ls -l new-boot.img

# flash (A-only device, boot partition is not mounted while running)
su -c 'dd if=new-boot.img of=/dev/block/by-name/boot bs=1M conv=fsync && sync'
```

Recovery, if it does not boot:

```bash
fastboot flash boot boot_current_magisk_RMX1821.img   # from any PC with platform-tools
```

## Backups (public, no auth)

Release: **https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/tag/backup-v1**

| Asset | sha256 |
|---|---|
| `boot_current_magisk_RMX1821.img` | `b5c65e5975f27b44bdfdccbf999b35b503d570b14cc014bb0962f102ba7b800c` |
| `dtbo_stock_RMX1821.img` | `1f180528f0167ec590cbb644490b19103db146a08d7f082cb60dac749fcbfe7f` |

`boot_current_magisk_RMX1821.img` is the current Magisk-patched boot (`.backup` and
`overlay.d` present in its ramdisk), not a factory-stock image.
