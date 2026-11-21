# Status, timeline and open questions

Last updated: 2026-11-21.

> **Keep this file current.** After every build, flash, or new finding, update the table at
> the top, append to the timeline, and adjust "verified vs. assumed". The repo is the only
> memory of this project — the phone may not survive it.

## Where things stand

| Item | State |
|---|---|
| Panel identified | done |
| Timing patch written | done |
| Backups dumped + published | done (`backup-v1` release) |
| CI pipeline | working (3 runs to green) |
| Kernel built successfully | yes, but the first green build was **4.14.282** — discarded |
| Kernel built at **4.14.235** | run `44923932703` in progress |
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
7. **Run `44921708014` — failed** at 8m09s, at the very end. `vmlinux` linked successfully; the
   DTB step died on `../scripts/dtc/dtc_overlay: Exec format error` — MTK ships that tool as
   a prebuilt **x86-64** binary. Fix: point `cmd_dtc` at kbuild's own host-arch `dtc`
   (`$(objtree)/scripts/dtc/dtc`).
8. **Run `44922739630` — SUCCESS** in 9m02s. Produced `Image.gz` (11 MB,
   sha256 `b63cdfb6…bdee8c`). **But** `out/include/config/kernel.release` was
   `4.14.282-M.A.P`, not `4.14.235-M.A.P`: the `M.A.P` branch has been updated since 2021.
   That image is ABI-adjacent to the wrong kernel and **must not be flashed**. The same run
   also revealed 31,256 bogus `-Wunknown-warning-option` lines from the GCC-style
   `-Werror-implicit-function-declaration` spelling.
9. **Run `44923932703` — in progress** (`variant=69`). Source pinned to commit
   `68d7a37a267743f12ac7d85054fea74564b8f7ea` (`SUBLEVEL = 235`, 2021-06-21). The build now
   aborts unless the release string is exactly `4.14.235-M.A.P`, and the artifact includes
   `.config` and `report.txt` (panel timing + `__crc_*` dump) for on-device verification.

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

- that commit `68d7a37a…` is the *exact* source of the running kernel — inferred from
  `SUBLEVEL = 235`, the `-M.A.P` localversion and the 2021-06-21 build timestamp. The
  `.config` diff against `/proc/config.gz` is the confirming check.
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

1. Wait for run `44923932703`; it must report `4.14.235-M.A.P` or it fails by design.
2. Download `Image-69hz`; compare the artifact `.config` with
   `~/rmx1821-backup/running.config` (from `/proc/config.gz`) — only `CONFIG_COMPAT_VDSO`
   should differ.
3. Repack against `boot_current_magisk_RMX1821.img`; run the three verification checks.
4. **Test without writing:** `fastboot boot new-boot-69hz.img` if LK supports it (confirm
   first with the current image — zero risk either way). Check the refresh overlay.
5. If `fastboot boot` is unsupported: upload the repacked image as a release asset, then
   flash 69 Hz with `dd`, release backup ready.
6. Only after 69 Hz is confirmed, build/test 72.
7. Record results here.

## Pre-flash testing (what is and isn't possible)

- **`fastboot boot <image>`** — loads into RAM, writes nothing. Best test: covers panel lock,
  HWC reporting, and module loading. Support varies by Realme LK build; try with the current
  image first.
- **Config diff** — running `/proc/config.gz` (5075 lines) vs artifact `.config`.
- **Module CRC check** — `report.txt` lists every `__crc_*` in `vmlinux`; compare against the
  `__versions` section of `/vendor/lib/modules/*.ko` (`wlan_drv_gen3.ko`, `fpsgo.ko`,
  `met.ko`, …).
- **Differential build** — build a stock-timing control and diff `objdump -d vmlinux`;
  differences should be confined to `lcm_get_params`.
- **kexec** — unavailable: `# CONFIG_KEXEC is not set`.
- **QEMU** — cannot emulate the MT6771 DSI panel; at most proves the image decompresses.
- **Recovery-mode boot** — the kernel comes from `boot`, not `recovery`, so this changes
  nothing.

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
