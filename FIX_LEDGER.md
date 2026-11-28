# Fix ledger: what was changed to get the first CI build running

Purpose: when the build works, move each fix to its proper home and drop the CI workaround.
"Now" = where the fix lives today. "Proper home" = where it should end up.
Update this file with every new fix.

## A. kitsunee02/miku-blossom (CI repo, branch main)

| Fix | Commit | Why | Proper home |
|---|---|---|---|
| Emptied `manifests/15.0/remove.xml` (restored abi-dumps, kernel prebuilts, developers/build, remoteexecution-client, mingw) | bf65486, efd167d | Soong fails on undefined modules (`vndk_abi_dump_zip`, `libwinpthread`, ...) | Stay in CI (nothing to move); keep removals out of the Hsin manifest too |
| `manifests/15.0/hsin.xml`: dropped local re-adds and the Car removal | f21f904 | Hsin `snippets/remove.xml` no longer removes them | Done in manifest; delete the CI file if empty |
| `manifests/15.0/hsin.xml`: **AOSP Launcher3 replaces the Hsin Launcher3** | 747f6bb | Hsin/Miku Launcher3 needs `org.lineageos.platform` (lineage-sdk) | Fork a Launcher3 without lineage deps into hsin-os, or add lineage-sdk. Decide before release |
| `patches/15.0/fixups.sh`: remove `hardware/google/pixel/kernel_headers` | f58bac8 | Duplicate `qti_kernel_headers` (also defined by vendor/hsin) aborts Kati | vendor_hsin: delete the Qcom stub `qti_kernel_headers` in `build/soong/Android.bp`, or Hsin manifest/patch. Re-check once Qcom is fully removed |
| `fixups.sh` Lineage removals (`hardware/lineage/interfaces/sensors`, `biometrics/fingerprint`, `power-libperfmgr`, `hardware/interfaces/biometrics/fingerprint/2.1/default`) | earlier | Conflict with the blossom device tree | Hsin manifest `remove-project`, if they are whole projects |
| `scripts/sync.sh`: fewer parallel jobs, retry/back-off on HTTP 429 | abb69d6 | Google git rate limit | CI only |
| `scripts/mem-guard.sh`, `build-stage.sh` (cgroup `memory.high`, memory trace), check/stage deadlines, concurrency | 74a9289, eb81e52, df364c1 | 15 GB runner thrashed during Soong; lost runner | CI only |
| `scripts/trim.py` + `manifests/15.0/trim.list` + hook in `prune.sh` | 71d5bf2 | Smaller Soong graph (removes only unreferenced dirs) | CI only. Review `trim-report.txt` for what stays |
| `scripts/find-dupes.py` + check.yml step | cc3f0d8 | Kati stops at the first duplicate module | CI only (debug aid) |

## B. hsin-os repos (branch Moon_fox)

| Repo | Fix | Commit | Why |
|---|---|---|---|
| manifest | `snippets/remove.xml` reduced to what Miku replaces | 18845f2 | The extra removals broke Soong |
| manifest | restored `platform/cts`, `hardware/google/aemu`, `gfxstream`; `vendor/settings` fork added | 4a85881 and earlier | undefined modules; Settings library |
| platform_build_soong | fsgen: pin `lib/` modules to 32-bit, `lib64/` to 64-bit | 54e3573 | packaging conflicts for blobs present in both |
| platform_build_soong | fsgen: route those modules to `multilib.lib32/lib64.high_priority_deps` | 2706052 | flat deps only pack the first target |
| platform_build_soong | `soong_build` sets GC percent and memory limit itself | b2131e8 | ninja starts it with `env -i`, so GOGC/GOMEMLIMIT never arrive |
| platform_frameworks_base | removed the empty 200dp QS header spacer | 7d5b5cf8a | leftover from the Miku banner |
| platform_build / vendor_hsin / settings / sepolicy / HsinSettings | the Miku to Hsin rebrand | earlier | see REBRAND_NOTES.md |

## C. Device and vendor trees (kitsunee02, branch hsin)

| Repo | Fix | Commit | Why |
|---|---|---|---|
| device_xiaomi_blossom | `RevampedFMRadio` package removed | f1d80d8 | non-existent module in PRODUCT_PACKAGES |
| device_xiaomi_blossom | `configs/cgroups.json` from the working A15 ROM; device.mk points to it | 1c4bef0, 3e2f733 | `system/core/.../cgroups_30.json` does not exist |

## D. Open questions to revisit
- Launcher3: AOSP in CI vs the Hsin fork (needs lineage-sdk).
- `qti_kernel_headers` ownership once Qcom is fully removed.
- Whether the `lib`/`lib64` fsgen fix is the right long-term shape (it is in the soong fork now).
- `HSIN_DEVICE` is not set anywhere; `ro.hsin.device` is empty.
- `version_util.mk:182: Argument missing` / "Device blossom not found" noise in the envsetup phase.
- Unforked Miku repos: bootable/recovery, packages/overlays/Themes (navbar overlay name in NavigationBarInflaterView).

## Launcher3 QuickStep: injectDoublePress
- Error: `SystemUiProxy is not abstract and does not override abstract method injectDoublePress(int)` then `injectLongPress(int)` (javac reports one at a time) (Launcher3QuickStepLib javac).
- Why: Hsin/Miku SystemUI's ISystemUiProxy.aidl added `injectDoublePress`; stock AOSP Launcher3 (swapped in for the Miku launcher) lacks it.
- Fix now: miku-blossom `patches/15.0/fixups.sh` inserts a forwarding method into `quickstep/.../SystemUiProxy.java` when missing.
- Proper home: a commit in a hsin-os fork of platform/packages/apps/Launcher3 (then drop the fixups block), or move to the Hsin Launcher decision.

## vendor sepolicy: unknown type iso9660
- Error: `device/mediatek/sepolicy_vndr/basic/non_plat/vold.te:11: unknown type iso9660` (vendor_sepolicy.cil.raw).
- Why: Jayedkhan800/android_device_mediatek_sepolicy_vndr (lineage-22) keeps `allow vold iso9660:filesystem unmount;`; AOSP 15 has no iso9660 type. LineageOS's own sepolicy_vndr dropped it.
- Fix now: miku-blossom fixups.sh deletes the line.
- Proper home: fork that repo into hsin-os (or move to the device tree) and delete the rule there.

## CI: keep going + resumable failures
- build-stage.sh passes `-k` (ninja keeps going, all errors in one pass); failed stages now pack out/ so a resume run continues from the failure point.

## OTA: update_dynamic_partitions failed (error 25) in OrangeFox
- Symptom: `assert failed: update_dynamic_partitions(op_list, unsparse_super_empty.img)` error 25 (OrangeFox R11.1, Redmi 9A dandelion). Super size matches BOARD_SUPER_PARTITION_SIZE; nothing mounted; op list normal.
- Why: our zip passed unsparse_super_empty.img (2-arg form, rewrites super metadata). The working Miku zip has no super_empty image (1-arg form). The image is built by default when dynamic partitions are on and PRODUCT_BUILD_SUPER_EMPTY_IMAGE is unset.
- Fix now: miku-blossom fixups.sh appends `PRODUCT_BUILD_SUPER_EMPTY_IMAGE := false` after PRODUCT_BUILD_SUPER_PARTITION in device/xiaomi/blossom/device.mk.
- Proper home: device tree device.mk (device_xiaomi_blossom, branch hsin), then remove the fixups block.

## Stage 12 module errors (Soong) — Lineage app dependencies
- `kotlin-parcelize-compiler-plugin` (Glimpse, Twelve): AOSP `external/kotlinc` lacks it → swapped to LineageOS `android_external_kotlinc` in `snippets/lineage.xml`. Proper home: manifest (already there).
- `org.lineageos.lib.phone` / `spn-info` (Dialer): copied minimal module into `vendor/hsin/lineage-lib`. Proper home: vendor/hsin (keep).
- Glimpse, Twelve, Jelly, Etar, ExactCalculator, Recorder added to `vendor/hsin/build/product/hsin_product.mk`.
## UI/branding
- Removed Hsin Home Taskbar setting (vendor/hsinsettings); gesture handle radius 3dp / width 120dp (frameworks/base SystemUI dimens); Miku effect sounds replaced (frameworks/base data/sounds).
