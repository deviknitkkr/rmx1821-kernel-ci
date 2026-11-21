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
| Kernel built at **4.14.235** | yes — run `44923932703`, release `4.14.235-M.A.P` |
| Boot image repacked | yes — `boot-69hz-magisk.img`, 32 MiB, sha256 `7c5975d4…4d568bd` |
| Published for testing | release **`boot-69hz-v1`** (public) |
| `fastboot boot` actually loads the image | **unverified** — the backup test couldn't prove it (byte-identical to the boot partition); marker test still pending |
| First 69 Hz attempt (RAM) | fell back to stock (`bootreason=lk_crash`); boot partition unchanged |
| 69 Hz image test | **bootloop** — user rolled back; `boot` is back to `b5c65e59…` |
| Stock-timing control | **building**: run `44932757133` (`variant=stock`, no vfp/PLL change) |
| Diagnostic marker images | published in release **`ramboot-diag-v1`** |
| Flashed permanently | no — rolled back |

The boot partition is restored to the backup (`sha256 b5c65e59…`). The phone runs stock
`4.14.235-M.A.P` (`#34`, clang 9.0.3, 2021-06-21) at 60 Hz. The 69 Hz kernel bootlooped
before Android and left no pstore log.

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
10. **Run `44923932703` — SUCCESS** in 8m27s, pinned source. `kernel.release` is exactly
    **`4.14.235-M.A.P`**; `report.txt` shows `vertical_frontporch = 16`, `PLL_CLOCK = 360`
    (the 365 line is the inactive command-mode branch); `Image.gz` sha256
    `0c95d4f0…d9f1f8`. The same run revealed two cosmetic bugs (31k unknown-warning lines
    from the demoted `-Werror=unknown-warning-option`; `.config` dropped as a dotfile), both
    fixed.
11. **Local repack (69 Hz).** `magiskboot` v31.1, base = the live Magisk boot. Kernel blob
    10,322,730 → 10,976,604 bytes; ramdisk **byte-identical** (`44bb37db…d855`),
    `.backup`/`overlay.d` intact; image exactly 32 MiB. Raw kernel sha256 `4d20949e…7f4071`.
    Published as release **`boot-69hz-v1`**, sha256 `7c5975d4…4d568bd`.
12. **`fastboot boot` believed working** — the backup image appeared to RAM-boot via the Pixel.
    Caveat: the backup is byte-identical to the `boot` partition, so that test could not
    distinguish a real RAM boot from a normal boot. Entry 13 resolves it.
13. **First 69 Hz RAM-boot attempt did not take.** Overlay still read 60. Verification:
    live `uname -v` = `#34 … clang 9.0.3 … Mon Jun 21 23:07:26 IST 2021`; live
    `CONFIG_COMPAT_VDSO=y` (our build has it off); boot partition sha256 still
    `b5c65e59…` (nothing written); cmdline says `androidboot.bootreason=lk_crash`; uptime
    had reset. So the running kernel was stock. Either the wrong file was booted or this
    LK rejected/ignored the uploaded image. Release **`ramboot-diag-v1`** adds a
    stock-kernel image with `ramtest=stock_marker` in the cmdline to settle it: if the
    marker shows up in `/proc/cmdline`, the upload is honored; if not, `fastboot boot` is
    a no-op on this LK and only a real flash can test the kernel.
14. **69 Hz image tested — bootloop; rollback clean.** The 69 Hz kernel was installed/booted and
    bootlooped before Android. The user restored the backup; the live kernel is stock `#34`,
    and `boot` hashes to `b5c65e59…`, so the rollback is verified. The failed boot left no
    kernel log in pstore (`console-ramoops` still ends with the previous clean `reboot
    bootloader`), which suggests a hang/panic before pstore or console init — consistent with
    display init hanging on the new panel timing, but not yet proven. A **stock-timing
    control** (`variant=stock`, same clang 18 pipeline, vfp/PLL untouched) is queued as run
    `44932757133`: if it boots, the timing change is the cause; if it also bootloops, the
    build/toolchain is.

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
- after the failed 69 Hz attempt the `boot` partition still hashes to `b5c65e59…`
- the fallback boot reported `androidboot.bootreason=lk_crash`; live kernel stayed stock
  (`#34`, clang 9.0.3, 2021) and `CONFIG_COMPAT_VDSO=y`
- the 69 Hz image **bootlooped**; rollback restored the exact backup (`b5c65e59…`)
- the bootloop left no pstore kernel log

**Assumed, not yet observed**

- that `fastboot boot` on this LK actually loads the uploaded image (the backup test was
  inconclusive — same bytes as the partition; the `stock_marker` image settles it)
- that the bootloop is caused by the panel timing — the **stock-timing control** (run
  `44932757133`) separates timing from the clang-18 build/toolchain

- that commit `68d7a37a…` is the *exact* source of the running kernel — inferred from
  `SUBLEVEL = 235`, the `-M.A.P` localversion and the 2021-06-21 build timestamp. The
  `.config` diff against `/proc/config.gz` is the confirming check.
- that a conservative vfp (200, 100, 60, 36) is accepted by the ILI9881 controller
- that MTK HWC reports the new rate to SurfaceFlinger
- that the rebuild produces a bootable image with identical vermagic
- that vendor kernel modules (Wi-Fi, touch, camera, GPU) still load — they should, since
  `CONFIG_LOCALVERSION`, `CONFIG_MODVERSIONS` and the defconfig are unchanged, but this is
  the most likely non-panel failure

## Open questions and how to resolve them

### 1. Is the bootloop caused by the panel timing?

The 69 Hz image bootlooped; the rollback is clean. First test the **stock-timing control**
(run `44932757133`, `variant=stock`): same clang 18 pipeline, `vfp`/PLL untouched.

- **Control boots** → the timing change is the cause. Step conservatively:
  `vfp = 200` (~61.2 Hz), `100` (~64.9), `60` (~66.6), `36` (~67.6), `16` (~68.5). Find
  the highest value that boots, then back off one step.
- **Control also bootloops** → the build/toolchain is the cause (config drift, clang 18,
  image format). Fix that before any more timing tests: diff the artifact `.config` against
  `/proc/config.gz`, check the kernel image/boot header, and consider the 2021 clang or the
  Android toolchain.

Do not assume the panel is at fault until the control boots.

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

1. **Finish the stock-timing control** (run `44932757133`). Download `Image-stockhz` (kernel,
   `.config`, `report.txt`) and `boot-stockhz-magisk` (repacked boot). The `.config` is the
   first chance to diff against the phone's `/proc/config.gz`.
2. Record the test details the user has not yet given: was the 69 Hz image **flashed** or
   **`fastboot boot`ed**, which file, what the bootloop looked like, and whether the
   `stock_marker` image ever showed `ramtest=stock_marker`.
3. Test the stock control (prefer `fastboot boot` if the marker proved it; otherwise flash
   and keep `backup-v1` ready):
   - **Boots** → timing is the cause. Build conservative variants (`vfp = 200`, then `100`,
     `60`, `36`) and test each. The workflow currently has `stock`/`69`/`72`; add a `vfp`
     input or more variants for this.
   - **Bootloops** → build/toolchain is the cause. Compare `.config` with `/proc/config.gz`,
     confirm the boot image header/format, and try the older clang or Android toolchain.
4. Only after a conservative timing boots, step toward `vfp = 16` (~68.5 Hz). Do not attempt
   `72` until `68.5` is stable.
5. If `fastboot boot` is still unverified, the `stock_marker` test is still the zero-risk way
   to settle it; otherwise every kernel test is a flash + rollback cycle.

## Pre-flash testing (what is and isn't possible)

- **`fastboot boot <image>`** — loads into RAM, writes nothing. Best test: covers panel lock,
  HWC reporting, and module loading. **Not yet proven on this LK**: the backup test was
  inconclusive because the backup equals the boot partition. The `stock_marker` image
  (release `ramboot-diag-v1`) is the definitive check.
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
