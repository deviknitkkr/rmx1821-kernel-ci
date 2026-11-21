# Device and display research

## Hardware / software identity

| | |
|---|---|
| Model | Realme 3, `RMX1821` (India). `RMX1825` is the same board in other regions |
| SoC | MediaTek MT6771 "Helio P60": 4× Cortex-A73 + 4× Cortex-A53, Mali-G72 MP3 |
| RAM / storage | 3 GB / 32 GB eMMC, ~21 GB `/data` |
| Display | 6.22" 720×1520 IPS LCD, 320 dpi, 60 Hz, no HDR, no VRR |
| Ports | micro-USB (OTG capable), no NFC |
| ROM | unofficial LineageOS 18.1: `lineage_RMX1821-userdebug 11 RQ3A.210605.005 eng.maanus.20210621.225552 test-keys` |
| Build id | `18.1-20210621-UNOFFICIAL-RMX1821` |
| Kernel | `4.14.235-M.A.P` (`#34 SMP PREEMPT Mon Jun 21 23:07:26 IST 2021`) |
| Root | Magisk, `su -c id` → `uid=0(root) ... u:r:magisk:s0` |
| Bootloader | unlocked, `androidboot.verifiedbootstate=orange` |
| Partition layout | A-only, **no slots**; `boot`, `dtbo`, `recovery`, `userdata`, `metadata`, `nvram`, `persist` … all separate |

Boot partition: `33,554,432` bytes (32 MiB).
DTBO partition: `16,777,216` bytes (16 MiB).

## How the panel is identified at boot

`/proc/cmdline` (abridged):

```
lcm=1-ili9881_qc_inx-fps=5994  bootopt=64S3,32N2,64N2
buildvariant=userdebug  androidboot.verifiedbootstate=orange
```

- `lcm=1-...` selects **index 1** from a multi-panel table (the board was also sold with an
  NT35595 FHD panel, whose driver is still compiled in).
- `fps=5994` is the bootloader telling the kernel 59.94 Hz.
- `/proc/device-tree/chosen/atag,videolfb-fps` contains bytes `00 00 17 70` → `0x1770` =
  6000 when byte-swapped, i.e. 60.00 Hz for the boot logo framebuffer.
- `/proc/device-tree/chosen/atag,videolfb-lcmname` reads
  `nt35595_fhd_dsi_cmd_truly_nt50358_drv` — **stale/default**, not the active panel. Trust
  the `lcm=` cmdline, not this property.

## Proof that there is no runtime control

1. **Android sees exactly one mode**

   `dumpsys display`:

   ```
   DisplayDeviceInfo{... 720 x 1520, ... supportedModes [{id=1, width=720, height=1520, fps=60.0}] ...}
   mActiveModeId=1   mDefaultModeId=1
   ```

   `dumpsys SurfaceFlinger`:

   ```
   primary range: [0.00 60.00]
   present offset: 0 ns   VSYNC period: 16666667 ns
   ```

   SurfaceFlinger has no second config to switch to, so `setFrameRate()` cannot help.

2. **Legacy MTK display stack, no DRM atomic modeset**

   ```
   CONFIG_MTK_FB=y
   (no CONFIG_DRM_MTK)
   ```

   `/sys/kernel/debug/disp/` exposes only `dump` and `lowpowermode`.
   `/sys/class/graphics/fb0/modes` = `U:720x1520p-0`.

3. **The panel driver has no fps callback**

   `drivers/misc/mediatek/video/mt6771/videox/disp_lcm.c` provides
   `disp_lcm_adjust_fps()`, which calls `lcm_drv->adjust_fps` if present. The ILI9881 driver
   does **not** define `adjust_fps`, so MTK's dynamic-FPS path is inert.

Physical conclusion: the scan rate is fixed by the compiled LCM timing table.

## The panel driver

Path:

```
drivers/misc/mediatek/lcm/ili9881_hd_dsi_vdo_qc_inx_zal1810/ili9881_hd_dsi_vdo_qc_inx_zal1810.c
```

Key facts from that file:

```c
#define LCM_DSI_CMD_MODE   0        /* video mode */
#define FRAME_WIDTH      (720)
#define FRAME_HEIGHT    (1520)

params->dsi.mode                    = SYNC_PULSE_VDO_MODE;
params->dsi.LANE_NUM                = LCM_THREE_LANE;
params->dsi.data_format.format      = LCM_DSI_FORMAT_RGB888;
params->dsi.packet_size             = 256;

params->dsi.vertical_sync_active    = 4;
params->dsi.vertical_backporch      = 8;
params->dsi.vertical_frontporch     = 236;   /* <-- the padding */
params->dsi.vertical_active_line    = FRAME_HEIGHT;

params->dsi.horizontal_sync_active  = 8;
params->dsi.horizontal_backporch    = 40;
params->dsi.horizontal_frontporch   = 40;
params->dsi.horizontal_active_pixel = FRAME_WIDTH;

/* video branch */
params->dsi.PLL_CLOCK = 360;                 /* MHz */
```

`vdo` in the directory name means **video mode**: the panel slaves to the host's VS/HS and
pixel clock, so changing the total frame length or the PLL changes the real scan rate. This
is the good case. (A command-mode panel would need its internal frame-rate register changed
instead.)

## The arithmetic

```
htotal = 720 + 8 + 40 + 40          = 808
vtotal = 1520 + 4 + 8 + 236         = 1768
observed rate                       = 59.94 Hz
```

Because `fps ∝ PLL_CLOCK / (htotal · vtotal)`:

| vertical_frontporch | vtotal | PLL_CLOCK | expected refresh |
|---|---|---|---|
| 236 (stock) | 1768 | 360 | 59.94 Hz |
| 200 | 1732 | 360 | ~61.2 Hz |
| 100 | 1632 | 360 | ~64.9 Hz |
| 60 | 1592 | 360 | ~66.6 Hz |
| 36 | 1568 | 360 | ~67.6 Hz |
| **16** | **1548** | **360** | **~68.5 Hz** |
| 0 (max blanking) | 1532 | 360 | ~69.2 Hz |
| 16 | 1548 | 376 | ~71.5 Hz |

The CI labels are approximate: variant `69` uses vfp 16 and is actually **~68.5 Hz**;
variant `72` is **~71.5 Hz**. The theoretical blanking-only maximum (vfp 0) is ~69.2 Hz.
The `vtotal` includes `vsa = 4` and `vbp = 8`, so vfp 16 gives 1548, not 1532.

Blank**ing alone cannot reach 72**: total blanking is only 248 lines, and 1768/1.2 = 1473,
which is below the 1520 active lines. So 72 Hz requires the PLL bump as well.

Why 236 is suspicious: 1768 lines is 14% longer than needed for 60 Hz at a PLL that could
run ~8% faster. The padding looks like an artifact of hitting exactly 60 Hz with a coarse
PLL step (360 MHz), which implies headroom rather than a hard panel limit.

## The patch

```
- params->dsi.vertical_frontporch = 236;
+ params->dsi.vertical_frontporch = 16;

  /* variant 72 only */
- params->dsi.PLL_CLOCK = 360;
+ params->dsi.PLL_CLOCK = 376;
```

The CI applies these with `sed` (see `build.md`), so context drift cannot break the patch.

## DSI link budget

3 lanes, RGB888, PLL 360 → ~720 Mbps/lane class, DSI PHY limit is ~1 Gbps/lane.
PLL 376 raises that by 4.4%, still far below the limit. The 69 Hz variant does not change
the link rate at all.

## What is *not* known yet

- Whether the ILI9881/INX module actually locks at 69–72 Hz. a-Si IPS panels usually
  tolerate 72, but not guaranteed.
- Whether MTK's proprietary HWC still reports 60 Hz. If it does, SurfaceFlinger schedules
  60 fps into a faster panel → beat-pattern judder, which is **worse** than stock. See
  `status.md`.
