# Changelog — Nasty Kernel (c2s, Exynos 990, 4.19.325)

Custom root-hiding kernel for the Galaxy Note20 Ultra (SM-N986B, `c2s`) on BenlyROM One UI 8.5.
KernelSU-Next (manual VFS hooks) + SUSFS v2.2.0 + kernel-level anti-detection.

Versioning: each release is a **superset** of the one before — higher number = strictly more,
never different. So an older build is only ever a reduced-capability downgrade, not a variant.

---

## [Unreleased] — Planned: 1.2

**Theme: erase the manager-app fingerprint + allow a choice of compatible managers.**
A paired release — kernel **and** a rebuilt manager APK must ship together.

- **Custom-renamed manager (primary).** Rebuild the KSU-Next manager from the branch matching
  this kernel's driver (UAPI / version `33024`), change its `applicationId` off
  `com.rifsxd.ksunext` to a neutral name, strip identifying label/icon/strings, and sign it with
  our own key. The kernel's `EXPECTED_MANAGER_SIZE` / `EXPECTED_MANAGER_HASH`
  (`kernel/Kbuild`, currently `0x3e6` / `79e5901…21bf7` = the KSU-Next CI key) are repointed to
  **our** key's v2 signature-block size + SHA-256.
  - *Why:* Duck Detector's remaining manager hit is the literal string `com.rifsxd.ksunext` in
    process heap. KSU-Next 3.1.0+ exposes **no** hide/randomize-package toggle, so the only way
    the string stops existing is to change the package name at build time.
  - *Enabled by this kernel already:* `throne_tracker` identifies the manager by **signature,
    not package name**; package pin is off (signature-only); `CHANGE_MANAGER_UID` (10006) exists.
- **Multi-signature manager whitelist (secondary).** Patch `kernel/apk_sign.c`
  (`is_manager_apk` / `check_v2_signature`) to accept a **table** of `{size, hash}` pairs so more
  than one KSU-Next-compatible manager can be authorized without a rebuild.
  - *Scope limit:* this only covers managers that speak **this driver's UAPI** (the KSU-Next
    lineage). It does **not** enable the upstream KernelSU manager or SukiSU — those are a
    different driver / different UAPI and would require a separate kernel build + reflash, not an
    app-level choice.

**Known limits:** renaming the manager hides only the *manager-app* fingerprint. Driver/path
traces (`/data/adb/ksu`, SUSFS leftovers) remain the SUSFS/module layer's responsibility.

---

## [1.1] — 2026-11-26 · `de2e788`  *(current shipped release)*

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
