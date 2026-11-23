# BenlyKernel v4.0.4 + working SUSFS — c2s (Galaxy Note20 Ultra, SM-N986B)

Reproduction notes for the Path A build (KernelSU-Next manual hooks + SUSFS v2.2.0),
targeting BenlyROM OneUI 8.5 on the Exynos 990 `c2s`.

## Problem this fixes
benly's `BenlyROMkernel_exynos990` @ tag `4.0.4` ships SUSFS **compiled out**: the
`exynos9830_defconfig` lists `CONFIG_KSU_SUSFS_*` but the committed KernelSU-Next
submodule (upstream `c81cdf5`) does not *define* those symbols, so `make oldconfig`
drops them and `fs/susfs.c` is `#ifdef`'d away.

## Fix = correct KSU-side + 3 small reconciliation patches

1. Replace the decoy submodule with the SUSFS-enabled fork:
   `Android-Artisan/KernelSU-Next` @ tag `v3.1.0-legacy-susfs`
   (defines `CONFIG_KSU_SUSFS_*`, manual-hook capable, version code 33024,
    manager signature hash `79e590113c4c4c0c222978e413a5faa801666957b1212a328e46c00c69821bf7`).

   ```bash
   rm -rf KernelSU-Next
   git clone --depth 1 --branch v3.1.0-legacy-susfs \
     https://github.com/Android-Artisan/KernelSU-Next.git KernelSU-Next
   # symlink drivers/kernelsu -> ../KernelSU-Next/kernel already present in benly's tree
   ```

2. Apply the patches in this folder:
   - `01-kernel-side-susfs-fixes.patch` — adds `susfs_reorder_mnt_id()` (no-op; benly's
     v2.2.0 SUS_MOUNT hides via `hide_sus_mnts_for_non_su_procs`, not a tracked mnt_id
     list) and makes `fs/devpts/inode.c` declare `ksu_handle_devpts` `extern` instead of
     redefining it (the KSU-side defines it → was a duplicate-symbol link error).
   - `02-ksu-side-supercalls.patch` — no-ops the two **deprecated** commands
     `CMD_SUSFS_SET_ANDROID_DATA_ROOT_PATH` / `CMD_SUSFS_SET_SDCARD_ROOT_PATH`
     (their `susfs_set_i_state_on_external_dir()` was removed in benly's v2.2.0
     kernel-side; hiding is handled by `CMD_SUSFS_ADD_SUS_PATH[_LOOP]`).

   ```bash
   git apply 01-kernel-side-susfs-fixes.patch
   cd KernelSU-Next && git apply ../02-ksu-side-supercalls.patch && cd ..
   ```

   `susfs_is_sid_equal` / `susfs_zygote_sid` and `susfs_set_current_proc_umounted`
   are already provided by the KSU-side (`selinux/selinux.c`) and benly's
   `susfs_def.h` respectively — no patch needed.

## Build (this container)

Host tools: `flex cpio lz4` (installed via nix here; `apt install flex cpio lz4` elsewhere).
Toolchain: AOSP clang (benly auto-downloads clang-r450784d from android.googlesource.com).
That host is blocked in this environment, so this build used the system **clang-18**
LLVM toolchain instead — it compiled cleanly. For a byte-closer-to-official build,
use clang-14 (r450784d).

```bash
make LLVM=1 LLVM_IAS=1 ARCH=arm64 O=out -j"$(nproc)" \
     exynos9830_defconfig c2s.config ksu.config
make LLVM=1 LLVM_IAS=1 ARCH=arm64 O=out -j"$(nproc)"
```
Output: `out/arch/arm64/boot/Image` (+ DTBs under `out/arch/arm64/boot/dts/...`).

## Package (benly's committed toolchain)
Replicates `./build.sh -m c2s -k y` packaging: builds `dtb.img`, `dtbo.img`, the
ramdisk, then `mkbootimg` (header v2, pagesize 2048, base 0x10000000,
board SRPTB27C009KU, cmdline `androidboot.hardware=exynos990 loop.max_part=7`) →
`boot.img`, zipped with `update-binary`/`updater-script` into a recovery-flashable zip
that writes `boot` and `dtbo`.

## Flash + manager
- Flash the zip in TWRP/OrangeFox (writes `/dev/block/by-name/boot` + `dtbo`).
- Manager APK **must** match the embedded signature: Android-Artisan KernelSU-Next
  release `v3.1.0-legacy-susfs`.
- Userspace hiding stack: ReZygisk/ZygiskNext → Shamiko → (optional) Tricky Store.

## Verified in the built image
`4.19.325-cip119-st3-BenlyKERNEL-v4.0.4+`, Hook mode: Manual (KPROBES off),
SUSFS v2.2.0, KernelSU avc-spoof + kernel_umount handlers registered,
valid ANDROID! boot header v2.
