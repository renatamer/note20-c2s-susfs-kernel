# Note20 Ultra (c2s) — BenlyKernel v4.0.4 + working SUSFS

Custom kernel build for the **Exynos 990 Galaxy Note20 Ultra (SM-N986B, codename `c2s`)**
running **BenlyROM OneUI 8.5**, with **KernelSU-Next (manual hooks) + SUSFS v2.2.0**
actually compiled in.

## Why this exists
benly's upstream `BenlyROMkernel_exynos990` @ `4.0.4` ships SUSFS **compiled out** — the
defconfig lists `CONFIG_KSU_SUSFS_*` but the committed KernelSU-Next submodule doesn't
*define* those symbols, so `make oldconfig` silently drops them. This build fixes that by
using the SUSFS-enabled KSU fork and reconciling a few version skews, so the kernel-level
hiding is real.

## Contents
| Path | What |
|------|------|
| `prebuilt/BenlyKernel-v4.0.4-SUSFS-c2s.zip` | Recovery-flashable (TWRP/OrangeFox); writes `boot` + `dtbo` |
| `prebuilt/boot.img` | Raw boot image (header v2) |
| `prebuilt/dtbo.img` | Device-tree overlay image |
| `prebuilt/SHA256SUMS` | Checksums |
| `susfs-patches/` | The 2 reconciliation patches |
| `configs/` | defconfig + `c2s`/`ksu` fragments used |
| `REPRODUCE.md` | Full rebuild recipe |

## Flash
1. Back up your current `boot.img` first.
2. TWRP/OrangeFox → Install → `BenlyKernel-v4.0.4-SUSFS-c2s.zip` → reboot.
3. Install the **official KernelSU-Next Manager** → https://github.com/KernelSU-Next/KernelSU-Next/releases
   The kernel trusts the **official KernelSU-Next signing key** (`KSU_NEXT_MANAGER_HASH = 79e590…`,
   byte-identical to upstream), so the official app is recognized as manager. It is
   **signature-validated, not version-locked** — a current release grants root; a ~v3.1.x
   release best matches this built-in susfs v2.2.0 driver (driver version code `33024`).
   (No Android-Artisan-specific APK is needed; their `v3.1.0-legacy-susfs` is only a git tag.)
4. Userspace hiding: ReZygisk/ZygiskNext → Shamiko → (optional) Tricky Store.

## Build
See `REPRODUCE.md`. Base = `benly-binh24/BenlyROMkernel_exynos990` @ tag `4.0.4`;
KSU-side = `Android-Artisan/KernelSU-Next` @ tag `v3.1.0-legacy-susfs`
(commit `ba4422f0`); then apply `susfs-patches/`.

## Credits / license
Linux kernel (GPL-2.0). Based on benly-binh24's BenlyROM kernel and Android-Artisan's
KernelSU-Next + SUSFS integration; SUSFS by simonpunk. This repo redistributes GPL kernel
source changes as patches and compiled images for personal use.
