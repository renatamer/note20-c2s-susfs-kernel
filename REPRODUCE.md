# Nasty Kernel — reproduction recipe (c2s, Exynos 990, 4.19)

Build a Nasty Kernel for the Galaxy Note20 Ultra (SM-N986B, `c2s`) on BenlyROM One UI 8.5:
KernelSU-Next (manual hooks) + working SUSFS v2.2.0 + kernel-level SELinux anti-detection.

## 1. Base source + KSU driver
```bash
git clone --depth 1 --branch 4.0.4 https://github.com/benly-binh24/BenlyROMkernel_exynos990.git nasty
cd nasty
# the committed submodule is a decoy (no susfs); swap in the SUSFS-enabled KSU fork:
rm -rf KernelSU-Next
git clone --depth 1 --branch v3.1.0-legacy-susfs \
  https://github.com/Android-Artisan/KernelSU-Next.git KernelSU-Next
# drivers/kernelsu -> ../KernelSU-Next/kernel symlink is already in benly's tree
```

## 2. Apply the patches (this repo)
- `susfs-patches/01-kernel-side.patch` — susfs no-op reconciliation (`susfs_reorder_mnt_id`,
  the deprecated `set_i_state` callers), makes `fs/devpts/inode.c` declare `ksu_handle_devpts`
  `extern` (it's defined KSU-side), and sets the `NastyKernel-1.1` banner.
- `susfs-patches/02-ksu-hardening.patch` — the anti-detection work, applied inside `KernelSU-Next/`:
  - **SELinux `ksu_file` → random label** (`KERNEL_SU_FILE`). The patch ships a **placeholder**
    `REPLACE_WITH_RANDOM_LABEL` — **generate your own and substitute it before applying:**
    ```bash
    LABEL="$(tr -dc 'a-z' < /dev/urandom | head -c8)"; echo "my label: $LABEL"
    sed -i "s/REPLACE_WITH_RANDOM_LABEL/$LABEL/" susfs-patches/02-ksu-hardening.patch
    ```
  - **seqno-split fix** in `kernel/selinux/rules.c`: KSU's AVC reset mirrors a real policy reload
    (`++selinux_state.ss->latest_granting`) instead of passing `0`, so the status page's
    `policyload` stays consistent (no `app_zygote` seqno split).
  - **version pin** to `33024` in `kernel/Kbuild`.
  - no-op the two deprecated external-dir susfs commands in `supercalls.c`.
```bash
git apply susfs-patches/01-kernel-side.patch
( cd KernelSU-Next && git apply ../susfs-patches/02-ksu-hardening.patch )
```

## 3. Toolchain — clang-14 ONLY
benly auto-downloads AOSP clang-r450784d (clang-14). **A clang-18 build compiles cleanly but the
kernel HANGS at the boot logo** (silent miscompilation of this 4.19 tree) — confirmed on-device.
The verified build used **upstream LLVM clang 14.0.0**
(https://github.com/llvm/llvm-project/releases/tag/llvmorg-14.0.0); put its `bin/` first on PATH.
Host tools: `flex cpio lz4`.

## 4. Build
```bash
make LLVM=1 LLVM_IAS=1 ARCH=arm64 O=out -j"$(nproc)" exynos9830_defconfig c2s.config ksu.config
make LLVM=1 LLVM_IAS=1 ARCH=arm64 O=out -j"$(nproc)"
```
Verify: `strings out/arch/arm64/boot/Image | grep "Linux version"` → `NastyKernel-1.1`,
and your random label present while `ksu_file` is absent.

## 5. Package (benly's committed toolchain)
`./build.sh -m c2s -k y` style: build `dtb.img` + `dtbo.img`, the ramdisk, then `mkbootimg`
(header v2, pagesize 2048, base 0x10000000, board SRPTB27C009KU,
cmdline `androidboot.hardware=exynos990 loop.max_part=7`) → `boot.img`, zipped with
`update-binary`/`updater-script` (writes `boot` + `dtbo`).

## 6. Flash + manager
- TWRP/OrangeFox → Install the zip. Back up Boot + DTBO first.
- Manager: official KernelSU-Next (the kernel trusts the official signing key); `~v3.1.x` matches
  susfs v2.2.0. Version is pinned to 33024 to match.

## What the hardening achieves (vs Duck Detector)
- `u:object_r:ksu_file:s0` and `u:r:ksu:s0` → **INVALID** (renamed/absent).
- `app_zygote seqno split` → **gone** (`policyload` consistent).
- **Not** fixed (not kernel-level): unlocked bootloader (hardware); Xposed `xposed_data`/
  `xposed_file` contexts if you run an Xposed framework like Vector (userspace).
