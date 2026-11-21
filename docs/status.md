# Status, timeline and open questions

Last updated: 2026-11-21.

## Where things stand

| Item | State |
|---|---|
| Panel identified | done |
| Timing patch written | done |
| Backups dumped + published | done (`backup-v1` release) |
| CI pipeline | iterating (3rd run in progress) |
| Kernel built successfully | **not yet** |
| Boot image repacked | not yet |
| **Flashed to the phone** | **not yet — phone is still on stock kernel** |

Nothing on the device has changed. The phone still runs `4.14.235-M.A.P` at 60 Hz.

## Timeline

1. **Diagnosis.** Confirmed a single `720x1520 @ 60.00` mode, legacy MTK fb
   (`CONFIG_MTK_FB=y`), no DRM, no FPS callback in the panel driver. Ruled out every
   runtime switch (Android `setFrameRate`, `/d/disp/`, `adjust_fps`).
2. **Panel identified** from `/proc/cmdline`: `lcm=1-ili9881_qc_inx-fps=5994`,
   driver `ili9881_hd_dsi_vdo_qc_inx_zal1810`, video mode, `vfp = 236`, `PLL_CLOCK = 360`.
3. **Backups.** Dumped `boot` and `dtbo` as root, verified SHA-256, published as public
   release assets (`backup-v1`).
4. **CI repo created** (`deviknitkkr/rmx1821-kernel-ci`, public). Workflow builds
   `Image.gz` on `ubuntu-24.04-arm` and uploads it as an artifact.
5. **Run `44919352618` — failed** in ~30 s: `arm-linux-androideabi-gcc not found, check
   CROSS_COMPILE_ARM32`. Cause: `CC=clang` passed via `env:` is overridden by the kernel
   Makefile's `CC = $(CROSS_COMPILE)gcc`; GCC was used, and GCC needs the 32-bit toolchain
   for the compat vDSO. Fix: pass the toolchain as **make arguments**, and disable
   `CONFIG_COMPAT_VDSO`.
6. **Run `44919736371` — failed** at 5m37s in `sound/soc/mediatek`: clang 18 promotes
   `-Wstrict-prototypes` / `-Wdeprecated-non-prototype` to errors, because MTK's Makefiles
   add `subdir-ccflags-y += -Werror` and the top Makefile adds `-Werror=strict-prototypes`
   etc. Fix: demote `-Werror` → `-Wno-error` across all `Makefile`/`Kbuild`/`*.mk` before
   building.
7. **Run `44921708014` — in progress** (`variant=69`), started after the `-Werror` fix.

Check it with:

```bash
gh run list --repo deviknitkkr/rmx1821-kernel-ci --workflow build.yml --limit 5
gh run view <id> --repo deviknitkkr/rmx1821-kernel-ci --log-failed
```

## Verified vs. assumed

**Verified on the device**

- panel driver path, timing values, PLL, mode, lane count
- boot image is Magisk-patched (`.backup`, `overlay.d`, patched `init`)
- boot = 32 MiB, dtbo = 16 MiB, A-only layout
- SHA-256 of both backups
- kernel release string of the running kernel is exactly `4.14.235-M.A.P`

**Assumed, not yet observed**

- that cutting `vfp` to 16 is accepted by the ILI9881 controller
- that 69–72 Hz is stable (no flicker, no dropped lines)
- that MTK HWC reports the new rate to SurfaceFlinger
- that the rebuild produces a bootable image with identical vermagic
- that vendor kernel modules (Wi-Fi, touch, camera, GPU) still load — they should, since
  `CONFIG_LOCALVERSION`, `CONFIG_MODVERSIONS` and the defconfig are unchanged, but this is
  the most likely non-panel failure

## Open questions and how to resolve them

### 1. Does the panel lock at 69/72?

Observe: no flicker/rolling bands, no black screen, logo renders, home screen stable.

If it fails: restore boot, then try conservative steps in between —
`vfp = 60` (~66.6 Hz), `vfp = 36` (~67.6 Hz) — to find the edge. If even `vfp = 200`
works but 16 does not, the limit is near 60 and the project ends there (restore stock).

### 2. Does Android see the new rate?

```bash
dumpsys display | grep -E "supportedModes|mActiveModeId"
dumpsys SurfaceFlinger | grep -E "VSYNC period|primary range"
```

- Reports 69/72 → done, judge by feel.
- Still reports 60 → **HWC mismatch**. The panel scans faster while SurfaceFlinger still
  paces frames at 60, producing a beat pattern. That is worse than stock. Options, in order:
  1. Restore stock and stop (safe default).
  2. Find the code path that publishes the mode to HWC (`mtkfb.c` / `primary_display.c`
     mode list, or the `fps=5994` cmdline consumption) and patch the reported vrefresh to
     match. This is the known next hunt; the exact function has not been located yet.

### 3. Are modules still loadable?

```bash
su -c 'dmesg | grep -i "disagrees about version\|unknown symbol\|vermagic"'
```

Empty → fine. If not, the build drifted from the defconfig; rebuild with
`RMX1821_defconfig` untouched except `CONFIG_COMPAT_VDSO`.

## Next concrete steps

1. Wait for run `44921708014`; if it fails, read `--log-failed`, fix, push, re-run.
2. Download `Image-69hz`; confirm `kernel.release == 4.14.235-M.A.P`.
3. Repack against `boot_current_magisk_RMX1821.img`; run the three verification checks.
4. Upload the finished `new-boot-69hz.img` as a release asset so the Pixel can fetch it
   without auth and so the recovery path is one file.
5. Flash **69 Hz first** (no DSI link change). Test. Only then consider 72.
6. Record results here.

## Things deliberately *not* tried

- **Runtime refresh switching** — no interface exists at any layer. A kernel rebuild is the
  only lever.
- **Changing `htotal`** — would raise the pixel clock and shrink horizontal blanking; more
  invasive than `vfp`, no benefit.
- **Editing the `fps=5994` cmdline** — that value feeds the boot logo framebuffer and
  reported mode, not the panel's own timing generator. On its own it would just make the
  kernel *think* the panel is faster.
- **Command-mode conversion** — the panel is wired and strapped for video mode.
- **Overclocking the GPU/CPU** — unrelated to the stutter being chased; the stutter is
  60 vs 120 Hz perception, not dropped frames (gfxinfo showed 5.45% janky, 90th pct 13 ms).

## Baseline measurements (stock kernel, for comparison)

- `dumpsys gfxinfo`: 7833 frames, 5.45% janky, 90th percentile 13 ms, 95th 17 ms
- SurfaceFlinger `VSYNC period`: 16,666,667 ns
- Developer "Show refresh rate" overlay: steady 60, no drops

If 69 Hz works, expect the overlay to read 69 and janky% to stay similar (the win is
perceptual smoothness, not fewer missed frames).
