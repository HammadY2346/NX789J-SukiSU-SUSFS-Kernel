# RedMagic 10S Pro (NX789J) — stock-matched SukiSU Ultra kernel

A rebuild of the **exact stock GKI kernel** that ships on the RedMagic 10S Pro
(Android 16, firmware 11.0.11) with root support built in. Nothing else is changed.

**Kernel:** `6.6.92-android15-8-g3637f4904cf5` — Google GKI build 13944661, rebuilt from
Google's public source at the same commit, same toolchain (clang r510928), same config.

## Added
- SukiSU Ultra (built-in, manual hooks)
- SUSFS 2.3.0
- KPM support

## Differences from stock (verified against the stock kernel)
| | stock | this |
|---|---|---|
| Config | — | only `MODULE_SIG_PROTECT` off (lets Google-signed `system_dlkm` modules load under a non-Google build key) and `FRAME_WARN` (compile-time warning threshold) |
| Module symbol CRCs | — | 0 mismatches, all stock vendor modules load unchanged |
| Modules loaded | 477 | 476 (one demand-loaded module differs by capture time) |
| CPU / GPU frequencies, governors, thermal zones | identical | identical |
| 3DMark Wild Life Stress | 25 303 / 20 408 / 80.6 %, 52 °C | 25 395 / 20 507 / 80.8 %, 51 °C |

## Install
1. Unlocked bootloader. Firmware **11.0.11** only (other firmware ships a different stock kernel).
2. Back up your stock `boot.img`.
3. Flash `boot-stock-sukisu-v2.img` to **boot** on your active slot (EDL, or `fastboot flash boot_a boot-stock-sukisu-v2.img`).
   Keep the **stock `init_boot`** — this image replaces the whole boot partition; do not combine it with an LKM-patched init_boot.
4. Install the SukiSU Ultra manager from the project's CI builds (artifact `Manager`, UAPI 6).
5. Optional: Manager → SUSFS → Spoof uname → Release `6.6.92-android15-8-g3637f4904cf5-ab13944661-4k`, execution location post-fs-data.

**Rollback:** flash your stock `boot.img` back.

The image is a full 96 MB boot partition with a valid AVB hash footer. On this device `boot` is a
chained AVB partition, so a bare kernel image flashed over the stock partition reboots to the bootloader.

## Notes
- The RedMagic-specific drivers (fan, shoulder triggers, display, charging, RGB…) are nubia's binary
  modules in `vendor_boot` / `vendor_dlkm` and are untouched. This is only the Google GKI part, which is
  why it can be rebuilt and verified to match.
- Build steps: [KERNEL-RECIPE.md](KERNEL-RECIPE.md).

## Credits
- [SukiSU Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra) · [SUSFS](https://gitlab.com/simonpunk/susfs4ksu) (simonpunk) and the
  [ShirkNeko/susfs4ksu](https://github.com/ShirkNeko/susfs4ksu) GKI branches · [SukiSU_patch](https://github.com/SukiSU-Ultra/SukiSU_patch) (KPM)
- [Android Common Kernel](https://android.googlesource.com/kernel/common/) — Google

Use at your own risk. Not responsible for bricked devices, lost data or banned game accounts.
