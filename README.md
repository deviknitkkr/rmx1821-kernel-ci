# RMX1821 kernel — raised panel refresh

> **Current state (2026-11-21):** the 69 Hz image was tested and **bootlooped**; the user
> rolled back and the phone is back on the stock kernel. A **stock-timing control** (same
> clang 18 build, no vfp/PLL change) is building. Start with
> [`docs/status.md`](docs/status.md); do not re-flash the 69 Hz image blindly.

Builds `Image.gz` for the Realme 3 (`RMX1821`, MT6771, ILI9881 INX HD+ panel) with the
panel's vertical front porch reduced, optionally with a 4.4% DSI PLL bump.

| variant | vfp | PLL | expected refresh |
|---|---|---|---|
| `69` | 236 -> 16 | 360 (unchanged) | ~68.5 Hz |
| `72` | 236 -> 16 | 360 -> 376 | ~71.5 Hz |
| `stock` | 236 | 360 | 59.94 Hz (control) |

The names are approximate. With `vsa`+`vbp` = 12 lines, vfp 16 gives `vtotal = 1548`, so
blanking alone tops out at ~68.5 Hz (vfp 0 would be ~69.2 Hz).

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

## Testing a kernel build

The 69 Hz image **bootlooped**; do not re-flash it. Current test order:

1. **Stock-timing control** (run `44932757133`, `variant=stock`): confirm the repacked image
   boots at 59.94 Hz. If it bootloops too, the clang-18 build/toolchain is the problem; if it
   boots, the timing change is.
2. If the control boots, build conservative variants (`vfp = 200`, `100`, `60`, `36`) and
   step down. Do not jump back to `vfp = 16`.

Use RAM boot when possible — it writes nothing:

```bash
fastboot boot boot-ramtest-marker.img   # stock kernel + cmdline marker, safe
cat /proc/cmdline | tr ' ' '\n' | grep ramtest   # ramtest=stock_marker means it works
```

If `fastboot boot` is unproven on this LK, every test is a flash + rollback cycle. The
rollback is `fastboot flash boot boot_current_magisk_RMX1821.img` from the Pixel or a laptop.

Full verification and recovery detail: [`docs/flash-and-recovery.md`](docs/flash-and-recovery.md).

## Rebuilding from source

Trigger CI, then download the kernel artifact:

```bash
gh workflow run build.yml --repo deviknitkkr/rmx1821-kernel-ci -f variant=stock
gh run list --repo deviknitkkr/rmx1821-kernel-ci --workflow build.yml --limit 3
gh run download <run-id> --repo deviknitkkr/rmx1821-kernel-ci -n Image-stockhz -D ~/kernel-out
```

The same run also has a `boot-stockhz-magisk` artifact with a directly flashable `boot.img`
(the CI repack job). Validate it, then use the RAM-boot test above before flashing.

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
