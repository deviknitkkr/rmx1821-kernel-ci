# Build pipeline

## Why cloud build

The phone (Helio P60, 3 GB) takes 1.5–2.5 h and bakes itself at 70–85 °C. The Pixel 8
(Tensor G3, 8 GB) takes 30–45 min. A GitHub Actions arm64 runner takes ~20–30 min with no
local heat, and the artifact is a plain file that gets repacked on the phone.

The workflow only produces `Image.gz`. It never touches a device.

## Trigger a build

```bash
gh workflow run build.yml --repo deviknitkkr/rmx1821-kernel-ci -f variant=69
gh run list --repo deviknitkkr/rmx1821-kernel-ci --workflow build.yml --limit 5
gh run watch <run-id> --repo deviknitkkr/rmx1821-kernel-ci
```

`variant` is `69` (front porch only) or `72` (front porch + PLL 376).

Artifact name: `Image-69hz` / `Image-72hz`, containing `Image.gz`.

```bash
gh run download <run-id> --repo deviknitkkr/rmx1821-kernel-ci -n Image-69hz -D ~/kernel-out
```

> Artifacts of a public repo still require auth to download via the API. If the phone is
> gone, download from the Actions run page in a browser, or use the finished boot image
> once it is uploaded to the release.

## What the workflow does

1. `actions/checkout@v4` — gets the workflow + docs.
2. Installs `clang`, `lld`, `llvm`, `binutils`, `bc`, `bison`, `flex`, `dtc`, `python3`,
   `build-essential`. Creates `aarch64-linux-gnu-*` symlinks if the image lacks them.
3. Checks out the kernel at the **pinned commit** `68d7a37a…` (sublevel 235). The `M.A.P`
   branch tip is now 4.14.282, which would not match the phone's vendor modules.
4. Patches (see below).
5. Builds `Image.gz` with `make ... Image.gz` (no modules).
6. Uploads `Image.gz`, `.config` and `report.txt` as an artifact.

Runner: `ubuntu-24.04-arm` (native aarch64; free for public repos). If that label ever
disappears, switch to `ubuntu-24.04` and add `gcc-aarch64-linux-gnu` plus an explicit
`--target=`; nothing else changes.

## The three bugs found (do not reintroduce them)

### 1. `CC=clang` must be a *make argument*, not an environment variable

The kernel Makefile contains `CC = $(CROSS_COMPILE)gcc`. An environment variable loses to
that assignment, so the first build silently used native GCC and reported `cc-name=gcc`.
The symptom was:

```
arch/arm64/Makefile:86: *** arm-linux-androideabi-gcc not found, check CROSS_COMPILE_ARM32. Stop.
```

Correct form:

```bash
ARGS=(
  ARCH=arm64 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm
  OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump STRIP=llvm-strip
  CROSS_COMPILE=aarch64-linux-gnu-
  CLANG_TRIPLE=aarch64-linux-gnu
  "KCFLAGS=-Wno-error=..."
)
make O=out "${ARGS[@]}" RMX1821_defconfig
make O=out -j"$(nproc)" "${ARGS[@]}" Image.gz
```

`CLANG_TRIPLE` must be **without** the trailing dash. `CROSS_COMPILE` keeps it, because it
is a binutils prefix.

### 2. Compat vDSO needs a 32-bit ARM toolchain

`CONFIG_COMPAT_VDSO=y` pulls in `arch/arm64/kernel/vdso32/`, requiring
`CROSS_COMPILE_ARM32` (`arm-linux-androideabi-` in the defconfig).

Solution used here: disable it in the defconfig before configuring.

```bash
sed -i 's/^CONFIG_COMPAT_VDSO=y/# CONFIG_COMPAT_VDSO is not set/' \
  arch/arm64/configs/RMX1821_defconfig
```

This is a **leaf feature**: it changes no exported symbol, no struct layout, no vermagic.
`CONFIG_COMPAT=y` remains, so 32-bit apps still run; they just lose the 32-bit vDSO fast
path for `gettimeofday`/`clock_gettime`.

Alternative (keeps the config identical): install `gcc-arm-linux-gnueabi` +
`binutils-arm-linux-gnueabi` and pass
`CROSS_COMPILE_ARM32=arm-linux-gnueabi- CLANG_TRIPLE_ARM32=arm-linux-gnueabi`.

### 3. `-Werror` everywhere

`-Werror` arrives from two places:

- MTK driver Makefiles: `subdir-ccflags-y += -Werror` (e.g.
  `sound/soc/mediatek/Makefile`, `sound/soc/mediatek/common_int/Makefile`). These are
  appended *after* `KCFLAGS`, so `-Wno-error=foo` in `KCFLAGS` cannot override them.
- The top Makefile: `-Werror=strict-prototypes`, `-Werror=implicit-int`,
  `-Werror=incompatible-pointer-types`, `-Werror=designated-init`, `-Werror=date-time`.

The 2021 build used clang ~12, which did not flag `void f()` prototypes. clang 18 does, and
`-Werror` turns those into build failures.

Solution, applied before building:

```bash
grep -rl --include='Makefile' --include='Kbuild' --include='*.mk' -- '-Werror' . \
  | xargs -r sed -i 's/-Werror/-Wno-error/g'
```

This changes build strictness only. No generated code, no ABI, no vermagic.

### 4. Prebuilt x86-64 `dtc_overlay`

`scripts/Makefile.lib` normally calls `$(srctree)/scripts/dtc/dtc_overlay`, a **binary
committed to the repo and built for x86-64**. On the arm64 runner it fails at the very end
of the build:

```
/bin/sh: 1: ../scripts/dtc/dtc_overlay: Exec format error
make[2]: *** [scripts/Makefile.lib:321: arch/arm64/boot/dts/mediatek/mt6771.dtb] Error 126
```

kbuild already compiles its own `dtc` for the host architecture (`HOSTCC
scripts/dtc/dtc.o` appears early in the log) and it takes the same flags
(`-@ -O dtb -b 0`). Redirect the rule:

```bash
sed -i 's#$(srctree)/scripts/dtc/dtc_overlay#$(objtree)/scripts/dtc/dtc#' scripts/Makefile.lib
```

`objtree` is the `O=out` directory, so this resolves to `out/scripts/dtc/dtc`.

### 5. The `M.A.P` branch no longer matches the phone

The first successful build produced `4.14.282-M.A.P`. The phone runs `4.14.235-M.A.P`
(built 2021-06-21). Stable updates move `struct module`, exported symbol sets and CRCs, so
an image from the tip is not a safe drop-in. The fix is to pin the commit whose `Makefile`
says `SUBLEVEL = 235`:

```
68d7a37a267743f12ac7d85054fea74564b8f7ea
```

The workflow checks that commit out and the build step aborts unless
`out/include/config/kernel.release` is exactly `4.14.235-M.A.P`. This is also why the
artifact now includes `.config` and `report.txt` — the config can be diffed against the
phone's `/proc/config.gz` before anything is flashed.

### 6. `-Werror-implicit-function-declaration` (hyphen, not `=`)

The top Makefile contains this GCC-style spelling. A naive
`s/-Werror/-Wno-error/g` turns it into `-Wno-error-implicit-function-declaration`, which
clang does not understand, so **every** object emits one `-Wunknown-warning-option` —
31,256 lines in the log. Convert the hyphen form to the `=` form first:

```bash
sed -i -e 's/-Werror-\([a-zA-Z0-9_-]*\)/-Wno-error=\1/g' \
       -e 's/-Werror/-Wno-error/g' <files>
```

### Harmless warning

`Node.js 20 is deprecated ... actions/checkout@v4` — cosmetic, ignore.

## Local build (Pixel 8 or the phone itself)

Identical to the CI steps. On a native aarch64 host:

```bash
pkg install -y clang lld make git bc bison flex python dtc
git clone --depth=1 -b M.A.P \
  https://github.com/Maanush2004/android_kernel_realme_RMX1821 ~/kernel
cd ~/kernel

grep -rl --include='Makefile' --include='Kbuild' --include='*.mk' -- '-Werror' . \
  | xargs -r sed -i 's/-Werror/-Wno-error/g'
sed -i 's/^CONFIG_COMPAT_VDSO=y/# CONFIG_COMPAT_VDSO is not set/' \
  arch/arm64/configs/RMX1821_defconfig

F=drivers/misc/mediatek/lcm/ili9881_hd_dsi_vdo_qc_inx_zal1810/ili9881_hd_dsi_vdo_qc_inx_zal1810.c
sed -i 's/params->dsi.vertical_frontporch = 236;/params->dsi.vertical_frontporch = 16;/' "$F"

ARGS=(ARCH=arm64 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm
      OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump STRIP=llvm-strip
      CROSS_COMPILE=aarch64-linux-gnu- CLANG_TRIPLE=aarch64-linux-gnu)
make O=out "${ARGS[@]}" RMX1821_defconfig
make O=out -j7 "${ARGS[@]}" Image.gz
```

`-j7` on the Pixel (4×A715 + X3 + 2×A510), `-j3` on the Realme, `-j4` on the CI runner.

## Artifact checks before flashing

```bash
cat out/include/config/kernel.release      # must be 4.14.235-M.A.P
ls -l out/arch/arm64/boot/Image.gz         # ~10 MB
```

The release string must match exactly, otherwise vendor modules fail vermagic checks.
`CONFIG_LOCALVERSION="-M.A.P"` and `CONFIG_MODVERSIONS=y` are preserved by using the
unmodified `RMX1821_defconfig`, which is why the checks pass.
