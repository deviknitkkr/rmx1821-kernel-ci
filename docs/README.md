# RMX1821 panel refresh project — handoff

**Status: build pipeline working-in-progress; nothing has been flashed to the phone yet.**

This folder exists so that another person (or an AI assistant) can pick this up cold —
including on a completely different device — and either continue the work or recover the
phone if a flash goes wrong.

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

1. Read all four docs first; do not assume the flash has happened (it has not, as of the
   last entry in `status.md`).
2. Check the latest build:
   `gh run list --repo deviknitkkr/rmx1821-kernel-ci --workflow build.yml --limit 5`
3. Download the artifact, then follow `flash-and-recovery.md` **in order**. The verification
   step (kernel release string, `.backup` present, image ≤ 32 MiB) is not optional.
4. Never suggest `fastboot flash` on a partition other than `boot` (or `dtbo` if and only if
   the device tree changed).
5. If the phone bootloops, the recovery image is a public release asset — no GitHub auth
   required to download it.
6. Do not ask the user to re-run Magisk's "patch a file" on `Image.gz`; that is a bare
   kernel, not a boot image.

---

## Access and secrets

- The CI repo is **public**: `https://github.com/deviknitkkr/rmx1821-kernel-ci`
- Backups: release tag **`backup-v1`**
- On the phone, the GitHub token lives at `~/.cache/gh_token` and
  `~/.config/gh/hosts.yml` (scopes `repo`, `workflow`, `read:org`).

**If the phone is lost or compromised, revoke that token** at
`https://github.com/settings/tokens`. It is only used for this repo.
