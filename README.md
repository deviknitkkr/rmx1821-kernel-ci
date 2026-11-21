# RMX1821 kernel — raised panel refresh

> **Current state (2026-11-21):** the 69 Hz tagged image was flashed and **bootlooped** at
> the Realme logo; the user rolled back and the phone is back on stock. `fastboot boot` is
> **proven working** (stock marker passed). The **stock-timing control** is published as
> release **`stock-control-v1`**; test it via RAM boot. Start with
> [`docs/status.md`](docs/status.md); do not re-flash the 69 Hz image.

Builds `Image.gz` for the Realme 3 (`RMX1821`, MT6771, ILI9881 INX HD+ panel) with the
panel's vertical front porch reduced, optionally with a 4.4% DSI PLL bump.

| variant | vfp | PLL | expected refresh |
|---|---|---|---|
| `stock` | 236 | 360 | 59.94 Hz (control) |
| `61` | 200 | 360 | ~61.2 Hz |
| `65` | 100 | 360 | ~64.9 Hz |
| `66` | 60 | 360 | ~66.6 Hz |
| `68` | 36 | 360 | ~67.6 Hz |
| `69` | 16 | 360 | ~68.5 Hz |
| `72` | 16 | 376 | ~71.5 Hz |

The names are approximate. With `vsa`+`vbp` = 12 lines, vfp 16 gives `vtotal = 1548`, so
blanking alone tops out at ~68.5 Hz (vfp 0 would be ~69.2 Hz). The `stock` control is the
current test; step down the ladder only after it boots.

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

The 69 Hz tagged image **bootlooped**; do not re-flash it. `fastboot boot` is **proven
working** — the stock marker booted and `/proc/cmdline` contained `ramtest=stock_marker`.
Use RAM boot for every test; it writes nothing.

1. **Stock control** — release **`stock-control-v1`**:

   ```
   https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/download/stock-control-v1/boot-stock-control.img
   sha256 a990490ba29dbbbf9da17a2492272846f72f7fd0e7cd9b38a0da50307a3a89e9
   ```

   ```bash
   fastboot boot boot-stock-control.img
   uname -v   # expect #1 ... Sat Nov 21 18:04:50 UTC 2026 (clang 18)
   dumpsys display | grep -E "supportedModes|mActiveModeId"   # expect 60 Hz
   ```

   - **boots at 60 Hz** → the 69 Hz timing caused the bootloop; test the ladder
     `61` (vfp 200), `65` (100), `66` (60), `68` (36), `69` (16).
   - **bootloops** → the clang-18 build/toolchain is the cause; do not flash.

2. If the control boots, dispatch a variant, repack, and RAM-boot it; repeat down the
   ladder. Stop at the highest that boots, then back off one step.

3. Do not flash until a variant is proven. Rollback if ever needed:
   `fastboot flash boot boot_current_magisk_RMX1821.img` from `backup-v1` (the Pixel/laptop
   can do this even if Android won't boot).

Full verification and recovery detail: [`docs/flash-and-recovery.md`](docs/flash-and-recovery.md).

## Rebuilding from source

Trigger CI, then download the kernel artifact:

```bash
# after the stock control boots, the first ladder step is 61:
gh workflow run build.yml --repo deviknitkkr/rmx1821-kernel-ci -f variant=61
gh run list --repo deviknitkkr/rmx1821-kernel-ci --workflow build.yml --limit 3
gh run download <run-id> --repo deviknitkkr/rmx1821-kernel-ci -n Image-61hz -D ~/kernel-out
```

The same run also has a `boot-61hz-magisk` artifact with a directly flashable `boot.img`
(the CI repack job). Validate it, then use the RAM-boot test above before flashing. The
control is already published, so no rebuild is needed for it.

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
