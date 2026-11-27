# Nasty Kernel — Galaxy Note20 Ultra (c2s), Exynos 990

Custom **4.19.325** kernel for the **Exynos 990 Galaxy Note20 Ultra (SM-N986B, codename `c2s`)**
running **BenlyROM (One UI 8.5)**, focused on **working root hiding**:

**KernelSU-Next (manual hooks) + working SUSFS v2.2.0 + kernel-level anti-detection
(SELinux + selective uname + a renamable, multi-manager driver).**

Current build: **Nasty Kernel 1.2** · built with **clang-14** · KSU driver version **33024**.

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
| **Selective uname spoof** · 1.2 | ✅ third-party apps (uid ≥ 10000) see a stock decoy; root/system/Settings see the real `Nasty-Kernel-1.2` |
| **`/proc/version` leak** · 1.2 | ✅ **closed** — it read `utsname()` directly and leaked the real kernel name; now uid-gated like `uname` |
| **Multi-signature manager** · 1.2 | ✅ slot 1 = stock KSU-Next key + slot 2 = your own key → run a **renamed** manager (erases the `com.rifsxd.ksunext` fingerprint) with **zero brick risk** |

Against **Duck Detector**, the kernel's entire KSU-attributable SELinux footprint is cleared
(`ksu_file`/`ksu` contexts INVALID, no seqno split) and the kernel name no longer leaks to apps.
Remaining detector dangers are **not** kernel-fixable: the **keybox/attestation** (TEE), the
**boot hash** (Bootloader), and any **Xposed framework** you run (e.g. Vector leaks
`xposed_data`/`xposed_file` + LSPosed signals — userspace).

## ⚠️ Anti-detection note (read this)
The SELinux type name is a **randomized label**, not `ksu_file`. The **published patch uses a
placeholder** (`REPLACE_WITH_RANDOM_LABEL`) on purpose — a *fixed, public* label is just a new
dictionary entry for detectors. **Generate your own** when you build (see `REPRODUCE.md`).
The **prebuilt binary contains one specific random label** (visible via `strings`); if that
matters, rebuild with a fresh label and/or keep this repo private.

**Manager signing key (slot 2):** the `04-*-ksu.patch` placeholders slot-2's cert size/hash
(`REPLACE_WITH_YOUR_MANAGER_CERT_*`) — generate your **own** signing key and bake its values.
The private keystore that signs your renamed manager is a **root credential** (any APK signed
with it is granted manager/root by the kernel) and is **deliberately not in this repo**. The
prebuilt trusts one specific slot-2 key; third parties can't use it (they lack the private key)
and fall back to the stock manager on slot 1.

## Contents
| Path | What |
|------|------|
| `prebuilt/NastyKernel-1.2-c2s.zip` | Recovery-flashable (TWRP/OrangeFox); writes `boot` + `dtbo`. **Boot confirmed on-device.** |
| `prebuilt/boot.img`, `dtbo.img`, `SHA256SUMS` | Raw images + checksums |
| `prebuilt/archive/1.1/` | Previous release (1.1) kept intact as its own recoverable build |
| `susfs-patches/01-kernel-side.patch` · `02-ksu-hardening.patch` | 1.0/1.1: susfs reconciliation + SELinux rename + seqno + version pin |
| `susfs-patches/03-nasty-1.2-kernel-side.patch` | 1.2: selective uname spoof + `/proc/version` leak fix + 1.2 banner |
| `susfs-patches/04-nasty-1.2-ksu.patch` | 1.2: multi-signature manager whitelist (slot-2 cert size/hash placeholdered) |
| `configs/` | defconfig + `c2s`/`ksu` fragments |
| `CHANGELOG.md` · `REPRODUCE.md` | release history · full rebuild recipe |

## Flash
1. **Back up your current `boot.img` + `dtbo`** first (TWRP: tick Boot + DTBO).
2. Must be on **BenlyROM One UI 8.5 (c2s)**.
3. TWRP/OrangeFox → Install → `NastyKernel-1.2-c2s.zip` → reboot.
4. Manager: the kernel trusts the **official KernelSU-Next** key (slot 1) — the official
   KSU-Next Manager (`~v3.1.x`, driver 33024) works out of the box. Slot 2 trusts **your own**
   key so you can run a renamed manager (see `REPRODUCE.md`).
5. **Activate the name hiding:** in the SUSFS module set a **Spoofed Kernel Version + Build**
   to a believable *stock* string (not "default"). Only then does the selective spoof engage —
   apps see the decoy, you/system see `Nasty-Kernel-1.2`.
6. Userspace hiding stack (reference): **sidex15 susfs4ksu-module** + ReZygisk/BreZygisk +
   **Tricky Store OSS** + HMA-OSS + **Specter** (keybox / PIF / cleanup). An Xposed framework
   (e.g. Vector) is optional and **loud** — it adds LSPosed + SELinux detections.

## Must build with clang-14
A **clang-18** build compiles but **hangs at the boot logo** (confirmed on-device). Use clang-14.

## Credits / license
Linux kernel (GPL-2.0). Based on benly-binh24's BenlyROM kernel, Android-Artisan's
KernelSU-Next + SUSFS, SUSFS by simonpunk. Redistributes GPL source changes as patches and
compiled images for personal use.
