# RMX1821 kernel — raised panel refresh

> **Current state (2026-11-21):** the 69 Hz kernel is built and published, but the first
> `fastboot boot` attempt fell back to the stock kernel. The `boot` partition is untouched.
> Start with [`docs/README.md`](docs/README.md#current-state--read-this-first) and run the
> marker test before flashing.

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

## Testing the 69 Hz image (no build needed)

Prebuilt: release **`boot-69hz-v1`** —
`boot-69hz-magisk.img`, sha256 `7c5975d481c4d1299367e254809afba2301c8d7adb8dec0e8e66b4ecf4d568bd`.

1. **Prove RAM boot first.** Download `boot-ramtest-marker.img` from release
   `ramboot-diag-v1` on the host (Pixel/laptop), then:

   ```bash
   fastboot boot boot-ramtest-marker.img
   cat /proc/cmdline | tr ' ' '\n' | grep ramtest
   ```

   `ramtest=stock_marker` means the upload is honored. **No output** means this LK ignores
   uploaded images; only a real flash can test the kernel.

2. If the marker appears, reboot and boot the tagged 69 Hz image:

   ```bash
   fastboot boot boot-69hz-tagged.img
   uname -v        # expect: #1 ... Sat Nov 21 2026   (clang 18)
   cat /proc/cmdline | tr ' ' '\n' | grep ramtest   # expect: ramtest=69hz_marker
   ```

3. Only after RAM boot is proven, flash. Full verification and the `dd` command are in
   [`docs/flash-and-recovery.md`](docs/flash-and-recovery.md). Keep the Pixel and
   `boot_current_magisk_RMX1821.img` ready for rollback.

## Rebuilding from source

Trigger CI, then download the kernel artifact:

```bash
gh workflow run build.yml --repo deviknitkkr/rmx1821-kernel-ci -f variant=69
gh run list --repo deviknitkkr/rmx1821-kernel-ci --workflow build.yml --limit 3
gh run download <run-id> --repo deviknitkkr/rmx1821-kernel-ci -n Image-69hz -D ~/kernel-out
```

The same run also has a `boot-69hz-magisk` artifact with a directly flashable `boot.img`
(the CI repack job). Validate it, then use the RAM-boot test above before flashing. Full
detail in [`docs/flash-and-recovery.md`](docs/flash-and-recovery.md).

Recovery, if it does not boot:

```bash
fastboot flash boot boot_current_magisk_RMX1821.img   # from the Pixel or a laptop
```

## Backups (public, no auth)

Release: **https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/tag/backup-v1**

| Asset | sha256 |
|---|---|
| `boot_current_magisk_RMX1821.img` | `b5c65e5975f27b44bdfdccbf999b35b503d570b14cc014bb0962f102ba7b800c` |
| `dtbo_stock_RMX1821.img` | `1f180528f0167ec590cbb644490b19103db146a08d7f082cb60dac749fcbfe7f` |

`boot_current_magisk_RMX1821.img` is the current Magisk-patched boot (`.backup` and
`overlay.d` present in its ramdisk), not a factory-stock image.
