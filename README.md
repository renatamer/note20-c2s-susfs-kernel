# Nasty Kernel — Galaxy Note20 Ultra (c2s), Exynos 990

Custom **4.19.325** kernel for the **Exynos 990 Galaxy Note20 Ultra (SM-N986B, codename `c2s`)**
running **BenlyROM (One UI 8.5)**, focused on **working root hiding**:

**KernelSU-Next (manual hooks) + working SUSFS v2.2.0 + kernel-level SELinux anti-detection.**

Current build: **Nasty Kernel 1.1** · built with **clang-14** · KSU driver version **33024**.

## Why this exists
It started as a fix for benly's `BenlyROMkernel_exynos990`, whose SUSFS was shipped
**compiled out** (the defconfig lists `CONFIG_KSU_SUSFS_*` but the committed KSU submodule
didn't define the symbols, so `make oldconfig` dropped them). From there it grew into a
root-hiding kernel that defeats the SELinux signatures most KSU setups leak.

## What's in it
| Layer | Status |
|---|---|
| **KernelSU-Next** (Android-Artisan `v3.1.0-legacy-susfs`) | ✅ built-in, manual VFS hooks (no kprobes) |
| **SUSFS v2.2.0** | ✅ actually compiled in (SUS_PATH/MOUNT/KSTAT/UNAME/CMDLINE/OPEN_REDIRECT/MAP) |
| **SELinux `ksu_file` type** | ✅ **renamed to a random label** → detector's `u:object_r:ksu_file:s0` probe returns INVALID |
| **`app_zygote` seqno split** | ✅ **fixed** — KSU's AVC reset mirrors a real policy reload (no `policyload=0`) |
| **Version pin** | ✅ KSU driver pinned to **33024** (matches official KSU-Next v3.1.0 manager) |

Against **Duck Detector**, the kernel's entire KSU-attributable SELinux footprint is cleared
(`ksu_file`/`ksu` contexts INVALID, no seqno split). Remaining detector dangers are **not**
kernel-fixable: the **unlocked bootloader** (hardware) and any **Xposed framework** you run
(e.g. Vector leaks `xposed_data`/`xposed_file` — userspace).

## ⚠️ Anti-detection note (read this)
The SELinux type name is a **randomized label**, not `ksu_file`. The **published patch uses a
placeholder** (`REPLACE_WITH_RANDOM_LABEL`) on purpose — a *fixed, public* label is just a new
dictionary entry for detectors. **Generate your own** when you build (see `REPRODUCE.md`).
The **prebuilt binary contains one specific random label** (visible via `strings`); if that
matters, rebuild with a fresh label and/or keep this repo private.

## Contents
| Path | What |
|------|------|
| `prebuilt/NastyKernel-1.1-c2s.zip` | Recovery-flashable (TWRP/OrangeFox); writes `boot` + `dtbo`. **Verified booting.** |
| `prebuilt/boot.img`, `dtbo.img` | Raw images |
| `prebuilt/SHA256SUMS` | Checksums |
| `susfs-patches/01-kernel-side.patch` | susfs reconciliation + `ksu_handle_devpts` extern + Nasty banner |
| `susfs-patches/02-ksu-hardening.patch` | **the hardening**: SELinux rename + seqno fix + version pin + supercalls no-op |
| `configs/` | defconfig + `c2s`/`ksu` fragments |
| `REPRODUCE.md` | full rebuild recipe |

## Flash
1. **Back up your current `boot.img` + `dtbo`** first (TWRP: tick Boot + DTBO).
2. Must be on **BenlyROM One UI 8.5 (c2s)**.
3. TWRP/OrangeFox → Install → `NastyKernel-1.1-c2s.zip` → reboot.
4. Manager: the kernel trusts the **official KernelSU-Next** signing key — install the
   official KernelSU-Next Manager (a `~v3.1.x` release matches this driver's susfs v2.2.0).
5. Userspace hiding stack: BreZygisk (Zygisk) + BRENE (susfs config) + HMA-OSS + Specter
   (PIF / keybox / detection cleanup) + TEESimulator (attestation).

## Must build with clang-14
A **clang-18** build compiles but **hangs at the boot logo** (confirmed on-device). Use clang-14.

## Credits / license
Linux kernel (GPL-2.0). Based on benly-binh24's BenlyROM kernel, Android-Artisan's
KernelSU-Next + SUSFS, SUSFS by simonpunk. Redistributes GPL source changes as patches and
compiled images for personal use.
