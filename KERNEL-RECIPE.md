# Reproducible build recipe — NX789J stock-matched SukiSU Ultra kernel

Target: RedMagic 10S Pro (NX789J), firmware 11.0.11 (Android 16), stock kernel
`6.6.92-android15-8-g3637f4904cf5-ab13944661-4k` = Google GKI build **13944661**.
Build host: Ubuntu on WSL2, ~60 GB free, `repo` + `git` + `python3` installed.

## 1. Sync the exact stock source
```bash
mkdir ~/gki-stock && cd ~/gki-stock
curl -fL -o manifest.xml "https://ci.android.com/builds/submitted/13944661/kernel_aarch64/latest/raw/manifest_13944661.xml"
sed -E 's/ (upstream|dest-branch)="[^"]*"//g' manifest.xml > stock.xml
repo init -u https://android.googlesource.com/kernel/manifest -b common-android15-6.6 --partial-clone --clone-filter=blob:none --no-use-superproject
cp stock.xml .repo/manifests/ && repo init -m stock.xml && repo sync -j8 --no-clone-bundle
git -C common log -1 --oneline     # expect 3637f4904cf5
```
(No `--depth`, no `--reference` — both break the pinned manifest.)

## 2. SukiSU Ultra (built-in) + SUSFS + KPM, inside `common/`
```bash
cd ~/gki-stock/common
curl -LSs https://raw.githubusercontent.com/SukiSU-Ultra/SukiSU-Ultra/main/kernel/setup.sh | bash -s builtin
git -C KernelSU checkout ce830530            # pinned builtin commit (UAPI 6)
# ports from the main branch needed by builtin on 6.6:
git -C KernelSU show origin/main:kernel/hook/patch_memory.h > KernelSU/kernel/hook/patch_memory.h
git -C KernelSU show origin/main:kernel/hook/arm64/patch_memory.c > KernelSU/kernel/hook/arm64/patch_memory.c
printf '#ifndef __KSU_H_KSUD_BOOT\n#define __KSU_H_KSUD_BOOT\n#include "ksud.h"\n#endif\n' > KernelSU/kernel/runtime/ksud_boot.h
sed -i 's|^#include "hook/lsm_hook.c"|#include "hook/lsm_hook.c"\n#ifdef __aarch64__\n#include "hook/arm64/patch_memory.c"\n#endif|' KernelSU/kernel/ksu.c
printf '\nKSU_NEW_DCACHE_FLUSH := $(shell grep -q __flush_dcache_area $(srctree)/arch/arm64/include/asm/cacheflush.h ; echo $$?)\nccflags-y += -DKSU_NEW_DCACHE_FLUSH=$(KSU_NEW_DCACHE_FLUSH)\n' >> KernelSU/kernel/Makefile

# SUSFS 2.3.0 (ShirkNeko/susfs4ksu, branch gki-android15-6.6)
git clone -b gki-android15-6.6 https://github.com/ShirkNeko/susfs4ksu ~/shirk
cp ~/shirk/kernel_patches/fs/susfs.c fs/ ; cp ~/shirk/kernel_patches/include/linux/susfs*.h include/linux/
patch -p1 -N -F3 < ~/shirk/kernel_patches/50_add_susfs_in_gki-android15-6.6.patch   # 0 failed hunks on 6.6.92

# config
sed -i '/^CONFIG_LIBNVDIMM=y$/a CONFIG_KPM=y' arch/arm64/configs/gki_defconfig
sed -i '/^CONFIG_KALLSYMS_ALL=y$/d' arch/arm64/configs/gki_defconfig          # KPM selects it; savedefconfig check fails otherwise
sed -i '/^CONFIG_MODULE_SIG_PROTECT=y$/d' arch/arm64/configs/gki_defconfig   # see README: lets Google-signed system_dlkm modules load
rm -f android/abi_gki_protected_exports_*; sed -i '/protected_exports_list/d' BUILD.bazel
```
KSU / SUSFS Kconfig default to y on the builtin branch; KPM is added above.

## 3. Build (stock uses LTO none)
```bash
cd ~/gki-stock
tools/bazel run --config=fast --config=stamp --lto=none //common:kernel_aarch64_dist -- --dist_dir=$PWD/out-dist
```
Verify against the phone: `scripts/extract-ikconfig out-dist/Image` diffed with `/proc/config.gz`
→ only `FRAME_WARN` and `MODULE_SIG_PROTECT`; module CRCs from `vmlinux.symvers` vs every
`.ko` in vendor_dlkm / vendor_boot → 0 mismatches.

## 4. KPM loader, boot image, AVB footer
```bash
# KPM: SukiSU_patch commit 547ae94 (15 Mar 2026). Do NOT use 0.13.9 (18 Oct 2026) — bootloops on this device.
cp out-dist/Image ~/kpm/Image && (cd ~/kpm && ./patch_linux)          # Image -> oImage, loader 3328
python3 tools/mkbootimg/mkbootimg.py --header_version 4 --kernel ~/kpm/oImage --cmdline '' -o boot.img
python3 avbtool.py add_hash_footer --image boot.img --partition_name boot --partition_size 100663296 \
  --algorithm SHA256_RSA4096 --key testkey_rsa4096.pem --rollback_index 1769904000 \
  --prop com.android.build.boot.os_version:15 --prop com.android.build.boot.security_patch:2026-02-01 \
  --prop "com.android.build.boot.fingerprint:nubia/NX789J/NX789J:15/AQ3A.240812.002/20260227.161747:user/release-keys"
```
The result must be exactly 100 663 296 bytes. `boot` is a **chained** AVB partition on this device:
flashing a smaller image leaves the stock footer behind → INVALID_METADATA → reboots to bootloader.

## 5. Flash
`boot` ← this image (EDL or fastboot), `init_boot` ← **stock**. Manager: SukiSU Ultra CI artifact `Manager`
(UAPI 6 must match the kernel). SUSFS → spoof uname release `6.6.92-android15-8-g3637f4904cf5-ab13944661-4k`.
