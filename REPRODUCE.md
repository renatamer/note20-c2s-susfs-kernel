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

# --- Nasty 1.2 ---
git apply susfs-patches/03-nasty-1.2-kernel-side.patch   # selective uname + /proc/version leak fix + 1.2 banner

# Generate YOUR OWN manager signing key, then bake its cert size/hash into slot 2:
keytool -genkeypair -alias mymanager -keyalg RSA -keysize 2048 -validity 10000 \
  -keystore my-manager.jks -storepass CHANGE_ME -keypass CHANGE_ME \
  -dname "CN=Manager, OU=x, O=x, L=x, ST=x, C=US"
keytool -exportcert -alias mymanager -keystore my-manager.jks -storepass CHANGE_ME -file cert.der
SIZE=$(wc -c < cert.der)                       # -> EXPECTED_MANAGER_SIZE_2
HASH=$(sha256sum cert.der | cut -d' ' -f1)     # -> EXPECTED_MANAGER_HASH_2
sed -i "s/REPLACE_WITH_YOUR_MANAGER_CERT_SIZE/$SIZE/; s/REPLACE_WITH_YOUR_MANAGER_CERT_HASH/$HASH/" \
  susfs-patches/04-nasty-1.2-ksu.patch
( cd KernelSU-Next && git apply ../susfs-patches/04-nasty-1.2-ksu.patch )
# KEEP my-manager.jks PRIVATE — any APK signed with it is granted manager/root by the kernel.
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
Verify: `strings out/arch/arm64/boot/Image | grep "Linux version"` → `NastyKernel-1.2`,
your random label present while `ksu_file` is absent, and both manager cert hashes present
(`strings Image | grep -c <slot1-hash>` and your slot-2 hash each return 1).

## 5. Package (benly's committed toolchain)
`./build.sh -m c2s -k y` style: build `dtb.img` + `dtbo.img`, the ramdisk, then `mkbootimg`
(header v2, pagesize 2048, base 0x10000000, board SRPTB27C009KU,
cmdline `androidboot.hardware=exynos990 loop.max_part=7`) → `boot.img`, zipped with
`update-binary`/`updater-script` (writes `boot` + `dtbo`).

## 6. Flash + manager
- TWRP/OrangeFox → Install the zip. Back up Boot + DTBO first.
- Manager: official KernelSU-Next on slot 1 (`~v3.1.x`, driver 33024, susfs v2.2.0) works out of
  the box — root never depends on the custom manager.
- **Name hiding is opt-in:** in the SUSFS module set a *Spoofed Kernel Version + Build* to a
  believable stock string (not "default"). The spoof is **global** — it engages only when a value
  is set, and then applies to every process so `uname`, `/proc/version` and `os.version` all agree.
  (A uid-gated/selective spoof is detectable: `os.version` is captured by the privileged Zygote
  and leaks the real name to apps, tripping a "kernel version spoofing" check.)

## 7. Renamed manager — erase the `com.rifsxd.ksunext` fingerprint (slot 2)
Build off-device (needs an Android SDK; not buildable in a minimal CI container):
1. Clone the **stock KernelSU-Next manager** source at the tag that reports driver **33024**
   (do not take a newer branch — UAPI drift breaks it).
2. Change `applicationId` off `com.rifsxd.ksunext` to a neutral name; optionally relabel/re-icon.
3. Build release, then sign **v2-ONLY** with your slot-2 keystore (KSU rejects v1 and v3 blocks):
   ```bash
   apksigner sign --ks my-manager.jks --ks-key-alias mymanager \
     --v1-signing-enabled false --v2-signing-enabled true \
     --v3-signing-enabled false --v4-signing-enabled false  app-release.apk
   ```
4. Install it, confirm root, then uninstall the stock `com.rifsxd.ksunext`. Slot 1 stays as a
   safety net.

## What the hardening achieves (vs Duck Detector)
- `u:object_r:ksu_file:s0` and `u:r:ksu:s0` → **INVALID** (renamed/absent).
- `app_zygote seqno split` → **gone** (`policyload` consistent).
- **Kernel name** no longer leaks to apps via `uname`/`/proc/version` (1.2, once the spoof is set).
- **Manager package fingerprint** removable via the renamed slot-2 manager (1.2).
- **Not** fixed (not kernel-level): keybox/attestation (TEE), boot hash (Bootloader), and any
  Xposed framework like Vector (`xposed_data`/`xposed_file` + LSPosed — userspace).
