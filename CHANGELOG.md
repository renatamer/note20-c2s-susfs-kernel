# Changelog — Nasty Kernel (c2s, Exynos 990, 4.19.325)

Custom root-hiding kernel for the Galaxy Note20 Ultra (SM-N986B, `c2s`) on BenlyROM One UI 8.5.
KernelSU-Next (manual VFS hooks) + SUSFS v2.2.0 + kernel-level anti-detection.

Versioning: each release is a **superset** of the one before — higher number = strictly more,
never different. So an older build is only ever a reduced-capability downgrade, not a variant.

---

## [Unreleased] — Planned: 1.3

- **SukiSU-Ultra driver swap** (tentative): a parallel, reflashable kernel that replaces the
  KernelSU-Next driver with SukiSU-Ultra. Not a manager toggle — different driver / UAPI. Open
  risk: SukiSU targets susfs 2.4.0 (no 4.19 drop-in) and KPM 5.10+, so it likely costs the
  susfs 2.2.0 hiding base that 1.1/1.2 rely on. Re-evaluated after 1.2 is stable.

---

## [1.2] — 2026-11-27  *(current shipped release)*

**Theme: real kernel name for the owner, a convincing decoy for apps, and a renamable,
multi-manager driver.** Binaries in `prebuilt/` (boot confirmed on-device, clang-14).
1.1 archived under `prebuilt/archive/1.1/`.

- **Selective uname spoof** (`kernel/sys.c`). The susfs uname spoof now applies **only to
  third-party apps** (`current_uid() >= 10000`); root/system (incl. Settings) see the **real**
  `Nasty-Kernel-1.2`. Active only when a spoof value is set in the SUSFS module — leave it
  "default" and nothing is spoofed (then the real name shows to everyone).
- **`/proc/version` leak closed** (`fs/proc/version.c`). `/proc/version` read `utsname()`
  directly and was **never spoofed** — it leaked the real kernel name to any reader (Duck
  Detector captured `…-NastyKernel-1.1+`). It now mirrors the same uid-gated spoof: apps get the
  decoy, root/system see the real name.
- **Multi-signature manager whitelist** (`kernel/apk_sign.c` + `Kbuild`). `is_manager_apk()`
  now accepts a **table** of manager certs: **slot 1** = stock KSU-Next CI key (your current
  manager keeps working — zero brick risk), **slot 2** = your own key. Package pin is off, so a
  **renamed, re-signed manager is accepted on signature alone** — this is what lets you erase the
  `com.rifsxd.ksunext` package fingerprint without ever losing root.
  - The published patch **placeholders** slot-2's size/hash (`REPLACE_WITH_YOUR_MANAGER_CERT_*`):
    generate your own signing key and bake its cert size + SHA-256, exactly like the SELinux label.
  - The **custom-renamed manager APK** is built off-device and signed **v2-only** with your
    slot-2 key (KSU rejects v1/v3 blocks); the kernel then trusts it with no rebuild.
- `CONFIG_LOCALVERSION="-NastyKernel-1.2"`.

**Known limits:** this does **not** touch the userspace Duck dangers — keybox (TEE), boot hash
(Bootloader), or an Xposed framework such as Vector (LSPosed + SELinux). Those remain the
attestation/module layer's job.

---

## [1.1] — 2026-11-26 · `de2e788`  *(superseded by 1.2; archived under `prebuilt/archive/1.1/`)*

**Theme: kernel-level SELinux anti-detection.** Built on 1.0; cut Duck Detector from 4 dangers
to 2. Binaries in `prebuilt/` (verified booting, clang-14).

- **SELinux `ksu_file` type → randomized label.** The detector probes the literal
  `u:object_r:ksu_file:s0`; with the type renamed, that context resolves **INVALID**. Published
  patch ships a `REPLACE_WITH_RANDOM_LABEL` placeholder on purpose — a fixed public label is just
  a new dictionary entry for detectors, so each builder generates their own.
- **`app_zygote` seqno-split fix** (`kernel/selinux/rules.c`). KSU's AVC reset now mirrors a real
  policy reload (`++selinux_state.ss->latest_granting` + `selinux_status_update_policyload`)
  instead of passing `0`, so the SELinux status page's `policyload` stays consistent — removes the
  seqno-split signature.
- **KSU driver version pinned to `33024`** (`kernel/Kbuild`) to match the official KSU-Next
  v3.1.0 manager (resolves the 33024-vs-33025 manager/kernel mismatch).
- **`supercalls.c`:** the two deprecated external-dir SUSFS commands no-op'd.
- `CONFIG_LOCALVERSION="-NastyKernel-1.1"`.

## [1.0] — rebrand + make SUSFS real  *(historical milestone; superseded by 1.1)*

**Theme: get a working SUSFS kernel + Nasty branding.** First Nasty-branded build, based on
benly `BenlyROMkernel_exynos990` @ tag `4.0.4`.

- **Fixed benly's compiled-out SUSFS.** The defconfig listed `CONFIG_KSU_SUSFS_*` but the
  committed KSU submodule was a decoy that never defined the symbols (so `make oldconfig` dropped
  them). Swapped it for **Android-Artisan `KernelSU-Next v3.1.0-legacy-susfs`** → **SUSFS v2.2.0
  actually compiled in** (SUS_PATH/MOUNT/KSTAT/UNAME/CMDLINE/OPEN_REDIRECT/MAP).
- **Toolchain: clang-14 only.** A clang-18 build compiles cleanly but **hangs at the boot logo**
  on this 4.19 tree (confirmed on-device). Verified build uses upstream LLVM clang 14.0.0.
- **Reconciliation fixes** for the KSU/SUSFS swap: `susfs_reorder_mnt_id` no-op stub;
  `fs/devpts/inode.c` declares `ksu_handle_devpts` `extern` (defined KSU-side); deprecated
  `set_i_state_on_external_dir` callers no-op'd.
- Rebranded from `BenlyKERNEL-v4.0.x`.

> **Note:** no standalone 1.0 binary is archived — 1.1 is a strict superset, so flashing 1.1
> gives everything 1.0 did plus the anti-detection work. 1.0 is documented here for history only.
