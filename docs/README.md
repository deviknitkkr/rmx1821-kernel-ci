# RMX1821 panel refresh project — handoff

**Status (2026-11-21): the 69 Hz tagged image was flashed and **bootlooped** at the Realme
logo; the user rolled back and the phone is back on the stock kernel. The stock marker proved
`fastboot boot` works. The **stock-timing control** is published as release
`stock-control-v1` — test it via RAM boot. See [status.md](status.md) for the live state.**

This folder exists so that another person (or an AI assistant) can pick this up cold —
including on a completely different device — and either continue the work or recover the
phone if a flash goes wrong.

## Current state — read this first

| Item | Value |
|---|---|
| Phone's `boot` partition | restored to the Magisk-patched backup, sha256 `b5c65e59…b800c` — **clean rollback** |
| Kernel actually running | stock `4.14.235-M.A.P #34`, clang 9.0.3, 2021-06-21, **60 Hz** |
| 69 Hz image | release **`boot-69hz-v1`**, `boot-69hz-magisk.img`, sha256 `7c5975d4…4d568bd` |
| 69 Hz test | **flashed `boot-69hz-tagged.img`**: Realme logo → reboot; rolled back via fastboot |
| Stock-timing control | **published**: release **`stock-control-v1`**, `boot-stock-control.img`, sha256 `a990490b…` |
| Diagnostic images | release **`ramboot-diag-v1`**: stock marker `026cb9a6…3ca55e`, 69 Hz tagged `4e6871ce…d9a360` |
| `fastboot boot` honors the upload | **verified** — stock marker booted; `/proc/cmdline` had `ramtest=stock_marker` |
| Flashed permanently | **no** — flashed once, rolled back |

The 69 Hz tagged image was flashed; the screen showed the Realme logo and rebooted. The
rollback restored the exact backup: live `uname -v` = `#34 … clang 9.0.3 … Mon Jun 21 2021`,
`boot` hash `b5c65e59…`. The stock marker image booted via `fastboot boot` and its cmdline
contained `ramtest=stock_marker`, so this LK does honor uploaded images — the 69 Hz kernel
itself crashed. The failed boot left no kernel log in pstore (`console-ramoops` still ends
with the previous clean `reboot bootloader`), suggesting a hang before pstore/console init.
That is consistent with display init hanging on the new panel timing, but not proven. The
stock-timing control decides it.

### Marker test (done: passed)

`fastboot boot boot-ramtest-marker.img` booted the stock kernel and `/proc/cmdline` contained
`ramtest=stock_marker`. So RAM boot works; future kernel tests need no flash. Keep this
procedure for re-checking, and **do not RAM-boot `boot-69hz-tagged.img`** — it bootloops.

Download on the **Pixel** (public, no auth):

```
boot-ramtest-marker.img  (stock kernel + marker)
https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/download/ramboot-diag-v1/boot-ramtest-marker.img
sha256 026cb9a6b590990da7806058c9a4a992bb91f265bd214f195d0b4d7b9d3ca55e
```

```bash
fastboot boot boot-ramtest-marker.img
# after Android is up:
cat /proc/cmdline | tr ' ' '\n' | grep ramtest
# expected and already observed: ramtest=stock_marker
```

### Test the stock control (next action)

```
boot-stock-control.img
https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/download/stock-control-v1/boot-stock-control.img
sha256 a990490ba29dbbbf9da17a2492272846f72f7fd0e7cd9b38a0da50307a3a89e9
```

```bash
fastboot boot boot-stock-control.img
uname -v        # expect: #1 SMP PREEMPT Sat Nov 21 18:04:50 UTC 2026 (clang 18)
dumpsys display | grep -E "supportedModes|mActiveModeId"
```

Expected: boots at 59.94 Hz with Wi-Fi, Magisk and vendor modules working. No flash.
If it bootloops, the clang-18 build/toolchain is the cause; if it boots, the 69 Hz timing
change is the cause and the conservative ladder (`61` → `65` → `66` → `68` → `69`) follows.

### Get these docs on another device

```bash
git clone https://github.com/deviknitkkr/rmx1821-kernel-ci
# or fetch a single file:
curl -L https://raw.githubusercontent.com/deviknitkkr/rmx1821-kernel-ci/main/docs/status.md
```

Public, no auth needed for reading docs or downloading release assets.

Read in this order:

| File | Contents |
|---|---|
| [device-and-display.md](device-and-display.md) | what the hardware is, what the display subsystem does, why it is stuck at 60 Hz, the exact patch and the arithmetic |
| [build.md](build.md) | GitHub Actions pipeline, the three bugs found, how to run a build, how to build locally |
| [flash-and-recovery.md](flash-and-recovery.md) | repack with Magisk, verify, flash on-device, and the full recovery procedure |
| [status.md](status.md) | timeline, run IDs, what is verified vs. assumed, open questions, next experiments |

---

## TL;DR

Realme 3 (`RMX1821`, MT6771 Helio P60, 3 GB) running unofficial LineageOS 18.1 with Magisk.
The panel is an Ilitek ILI9881 in **video mode** driven by MTK's legacy fb driver, hard-locked
to a single `720x1520 @ 60.00 Hz` mode. There is no runtime control.

The panel driver (`ili9881_hd_dsi_vdo_qc_inx_zal1810.c`) pads the frame with
`vertical_frontporch = 236` lines — about 14% blanking — apparently to land on exactly
60 Hz with a coarse 360 MHz DSI PLL. Cutting that porch to 16 lines raises the refresh to
**~68.5 Hz with no change to the DSI link rate**; additionally bumping the PLL to 376 MHz
gives **~71.5 Hz**. The CI labels `69` and `72` are approximate.

The change is two lines in a kernel source file. We build the kernel in GitHub Actions and
repack it into the phone's **existing Magisk-patched boot image**, so root and `/data` are
preserved.

No raised-refresh boot has succeeded yet. The panel may refuse the timing; the Android
HWC may keep reporting 60 Hz. Both outcomes and their handling are documented.

---

## Repository map

```
.github/workflows/build.yml   cloud build (workflow_dispatch: variant 69 or 72)
docs/                         this handoff
README.md                     short pointer
```

Kernel source is **not** vendored here. The workflow checks out a **pinned commit**:

```
https://github.com/Maanush2004/android_kernel_realme_RMX1821
commit 68d7a37a267743f12ac7d85054fea74564b8f7ea   (sublevel 235, 2021-06-21)
```

The `M.A.P` branch has moved on — its current tip reports `4.14.282`, while the phone runs
`4.14.235`. Building the tip would put the kernel in a different vermagic/CRC family from
the vendor modules, so the commit is pinned and the build fails if the release string is
anything other than `4.14.235-M.A.P`.

---

## If you are an AI assistant taking over

1. Read [`status.md`](status.md) first — it has the live state, evidence, and the latest
   timeline entry.
2. The phone is back on stock (60 Hz) and its `boot` partition is the exact backup. The
   69 Hz tagged image was **flashed** and bootlooped at the Realme logo; do not re-flash it.
   **`fastboot boot` is proven** (stock marker passed), so use RAM boot for tests — it writes
   nothing. Keep `backup-v1` and the Pixel/laptop available anyway.
3. Immediate next action: test **`stock-control-v1`** via RAM boot. If it boots at 60 Hz,
   the timing change caused the bootloop; if it also bootloops, the clang-18 build/toolchain
   is the cause. Do not skip this control.
4. If the control boots, build the conservative CI variants `61` (vfp 200), `65` (100),
   `66` (60), `68` (36), then `69` (16), and test each via RAM boot. Stop at the highest
   that boots, then back off one step. Do not jump straight back to `vfp = 16`.
5. The **marker test** already passed; no need to repeat it unless RAM boot stops working.
6. Rollback (if anything is ever flashed) is `fastboot flash boot
   boot_current_magisk_RMX1821.img` from `backup-v1`; the Pixel can do this even if Android
   won't boot. Keep the Pixel and cable available.
7. Never flash any partition other than `boot` (or `dtbo` if and only if the device tree
   changed). See `flash-and-recovery.md` for the full recovery procedure.
8. Do not ask the user to re-run Magisk's "patch a file" on `Image.gz`; that is a bare
   kernel, not a boot image.

---

## Access and releases

- The CI repo is **public**: `https://github.com/deviknitkkr/rmx1821-kernel-ci`
- Backups / rollback: release **`backup-v1`**
  (`boot_current_magisk_RMX1821.img`, sha256 `b5c65e59…b800c`)
- 69 Hz image: release **`boot-69hz-v1`** (`boot-69hz-magisk.img`, sha256 `7c5975d4…4d568bd`)
- Stock-timing control: release **`stock-control-v1`** (`boot-stock-control.img`, sha256
  `a990490b…3a89e9`)
- RAM-boot diagnostics: release **`ramboot-diag-v1`**
  (`boot-ramtest-marker.img` `026cb9a6…3ca55e`, `boot-69hz-tagged.img` `4e6871ce…d9a360`)
- On the phone, the GitHub token lives at `~/.cache/gh_token` and
  `~/.config/gh/hosts.yml` (scopes `repo`, `workflow`, `read:org`). To push or dispatch CI
  from a new device, create a fresh token there; the old one is only on the phone.

**If the phone is lost or compromised, revoke that token** at
`https://github.com/settings/tokens`. It is only used for this repo.
