# RMX1821 panel refresh project — handoff

**Status (2026-11-21): the 69 Hz kernel is built and published. The first RAM-boot attempt
fell back to the stock kernel; the `boot` partition is untouched. Next action: the marker
test in [flash-and-recovery.md](flash-and-recovery.md#step-0-prove-fastboot-boot-loads-the-uploaded-image).**

This folder exists so that another person (or an AI assistant) can pick this up cold —
including on a completely different device — and either continue the work or recover the
phone if a flash goes wrong.

## Current state — read this first

| Item | Value |
|---|---|
| Phone's `boot` partition | Magisk-patched image, sha256 `b5c65e59…b800c` — **untouched** |
| Kernel actually running | stock `4.14.235-M.A.P #34`, clang 9.0.3, 2021-06-21, **60 Hz** |
| 69 Hz image | release **`boot-69hz-v1`**, `boot-69hz-magisk.img`, sha256 `7c5975d4…4d568bd` |
| Diagnostic images | release **`ramboot-diag-v1`**: stock marker `026cb9a6…3ca55e`, 69 Hz tagged `4e6871ce…d9a360` |
| RAM boot | **unproven** — first attempt reset with `bootreason=lk_crash` and fell back to stock |
| Flashed permanently | **no** |

The first 69 Hz attempt booted the stock kernel. Evidence captured on the phone:

- live `uname -v` = `#34 … clang 9.0.3 … Mon Jun 21 23:07:26 IST 2021`
- our image's banner = `#1 … clang 18.1.3 … Sat Nov 21 16:27:40 UTC 2026`
- live `CONFIG_COMPAT_VDSO=y`; our build has it **off**
- `boot` still hashes to `b5c65e59…`, so nothing was written
- `/proc/cmdline` says `androidboot.bootreason=lk_crash`

So the 69 Hz kernel was never executed. Either the wrong file was booted, or this LK
ignored/crashed on the uploaded image. The marker image distinguishes those two cases.

### Marker test (do this before any flash)

Download on the **Pixel** (public, no auth):

```
boot-ramtest-marker.img  (stock kernel + marker)
https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/download/ramboot-diag-v1/boot-ramtest-marker.img
sha256 026cb9a6b590990da7806058c9a4a992bb91f265bd214f195d0b4d7b9d3ca55e

boot-69hz-tagged.img      (69 Hz kernel + marker)
https://github.com/deviknitkkr/rmx1821-kernel-ci/releases/download/ramboot-diag-v1/boot-69hz-tagged.img
sha256 4e6871ceac7bd82af7ec0bbe81bb0ed116a1cd3a7a8cd297d3965bb429d9a360
```

```bash
fastboot boot boot-ramtest-marker.img
# after Android is up:
cat /proc/cmdline | tr ' ' '\n' | grep ramtest
```

- **`ramtest=stock_marker`** → this LK honors uploaded images. Reboot, then
  `fastboot boot boot-69hz-tagged.img` and check `uname -v` (expect `#1 … Nov 2026`) plus
  `ramtest=69hz_marker`.
- **no output** → `fastboot boot` is a no-op on this LK. Only a real flash tests the kernel;
  `backup-v1` is the rollback (the Pixel can flash it even if Android won't boot).

The stock marker image is risk-free: stock kernel (`sha256 98e4a881…`), byte-identical
Magisk ramdisk, only the cmdline gains `ramtest=stock_marker`.

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
**~69 Hz with no change to the DSI link rate**; additionally bumping the PLL to 376 MHz
gives **~72 Hz**.

The change is two lines in a kernel source file. We build the kernel in GitHub Actions and
repack it into the phone's **existing Magisk-patched boot image**, so root and `/data` are
preserved.

Nothing above ~69 Hz has been observed yet. The panel may refuse the timing; the Android
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
   timeline entry. Do **not** assume the flash has happened; it has not.
2. The phone is currently stock (60 Hz) and its `boot` partition is untouched. The 69 Hz
   image is built but was not actually executed. Do not flash yet.
3. Run the **marker test** above. It is zero-risk and answers the only open question that
   matters right now: does this LK load an uploaded image?
4. If the marker appears, boot `boot-69hz-tagged.img` and verify `uname -v` (2026, clang 18)
   and `ramtest=69hz_marker`. If the 69 Hz kernel boots, check the overlay, `dumpsys
   display`, `dumpsys SurfaceFlinger`, Wi-Fi, and module errors.
5. If the marker does **not** appear, `fastboot boot` is a no-op on this LK. Only a real
   flash tests the kernel. The rollback is `fastboot flash boot
   boot_current_magisk_RMX1821.img` from `backup-v1`; the Pixel can do this even if Android
   won't boot. Keep the Pixel and cable available.
6. Never flash any partition other than `boot` (or `dtbo` if and only if the device tree
   changed). See `flash-and-recovery.md` for the full recovery procedure.
7. Do not ask the user to re-run Magisk's "patch a file" on `Image.gz`; that is a bare
   kernel, not a boot image.

---

## Access and releases

- The CI repo is **public**: `https://github.com/deviknitkkr/rmx1821-kernel-ci`
- Backups / rollback: release **`backup-v1`**
  (`boot_current_magisk_RMX1821.img`, sha256 `b5c65e59…b800c`)
- 69 Hz image: release **`boot-69hz-v1`** (`boot-69hz-magisk.img`, sha256 `7c5975d4…4d568bd`)
- RAM-boot diagnostics: release **`ramboot-diag-v1`**
  (`boot-ramtest-marker.img` `026cb9a6…3ca55e`, `boot-69hz-tagged.img` `4e6871ce…d9a360`)
- On the phone, the GitHub token lives at `~/.cache/gh_token` and
  `~/.config/gh/hosts.yml` (scopes `repo`, `workflow`, `read:org`). To push or dispatch CI
  from a new device, create a fresh token there; the old one is only on the phone.

**If the phone is lost or compromised, revoke that token** at
`https://github.com/settings/tokens`. It is only used for this repo.
