# RMX1821 kernel — raised panel refresh

Builds `Image.gz` for the Realme 3 (`RMX1821`, MT6771, ILI9881 INX HD+ panel) with the
panel's vertical front porch reduced, optionally with a 4.4% DSI PLL bump.

| variant | vfp | PLL | expected refresh |
|---|---|---|---|
| `69` | 236 -> 16 | 360 (unchanged) | ~69.2 Hz |
| `72` | 236 -> 16 | 360 -> 376 | ~72.3 Hz |

`69` does not change the DSI link rate at all — it only removes frame blanking.

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
gh run list --limit 3
gh run download <run-id> -n Image-69hz -D ~/kernel-out
```

Validate, repack **from the current (Magisk-patched) boot**, and flash:

```bash
cd /data/local/tmp/bootchk           # created by: magiskboot unpack boot_current.img
cp ~/kernel-out/Image.gz kernel
/data/adb/magisk/magiskboot repack boot_current.img new-boot.img

# sanity: still Magisk-patched, kernel gzip, size fits the 32 MiB partition
/data/adb/magisk/magiskboot unpack new-boot.img
/data/adb/magisk/magiskboot cpio ramdisk.cpio "exists .backup"
ls -l new-boot.img

# flash (A-only device, boot partition is not mounted while running)
dd if=new-boot.img of=/dev/block/by-name/boot bs=1M && sync
```

Recovery, if it does not boot:

```bash
fastboot flash boot boot_current.img     # from any PC with adb/fastboot
```

## Backups

`~/rmx1821-backup/boot_stock.img` is the current Magisk-patched boot (`.backup` and
`overlay.d` present in its ramdisk). Keep its sha256: `b5c65e59...b7b800c`.
