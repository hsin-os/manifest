# Hsin OS rebrand notes

Internal notes on what is renamed, what is intentionally left as "miku", and what is still open.

## Renamed (done)
| Was | Now | Where |
|---|---|---|
| `ro.miku.version/buildtype/maintainer`, `ro.system.miku.version` | `ro.hsin.*`, `ro.system.hsin.version` | set in vendor/hsin `config/versioning.mk`, read in Settings `HsinVersion/HsinType/MaintainerPreferenceController` (nothing in frameworks/base reads them) |
| `MikuVersion/MikuTypePreferenceController` | `HsinVersion/HsinTypePreferenceController` | Settings, `firmware_version.xml` |
| `com.miku.exsettings.*`, `MikuSettings` | `com.hsin.exsettings.*`, `HsinSettings` | vendor/HsinSettings, Settings (Android.bp, manifest, gateway, registry, proguard) |
| `MetricsEvent.MIKU` | `MetricsEvent.HSIN` (=3939) | frameworks/base metrics_constants.proto, vendor/HsinSettings |
| `device/miku/sepolicy` includes | `device/hsin/sepolicy` | device/hsin/sepolicy `*/sepolicy.mk` |
| `/data/miku_updates` label | `/data/hsin_updates` | sepolicy `file_contexts` |
| Settings symlink `MikuSettings` | `HsinSettings -> ../../../vendor/HsinSettings` | Settings |

## Intentionally still "miku" (not ours / would break)
- `com.miku.settings.preferences.*` (SystemSettingSwitchPreference, SecureSettingSwitchPreference, Utils, ...): provided by `vendor/settings` (Miku-UI fork in snippets/miku.xml). Rename only after forking that repo.
- `com.android.internal.util.miku.{ThemeUtils,FileUtils,MikuUtils,PropsUtils}` in frameworks/base (used by vendor/HsinSettings and Settings).
- vendor/hsin: `MIKU_*`/`PLATFORM_MIKU_*`/`TARGET_MIKU_*` make vars, `miku*Vars` soong namespaces, `miku_generator`, `prebuilts/tools-miku` path.
- Upstream copyright lines ("Miku UI", AOSP, AICP, crDroid ...) must stay.
- Settings preference keys backed by the settings DB (status_bar_show_lyric, lockscreen_media_metadata, smart_pixels_*, task_bar_enable, ...) are read by SystemUI/Launcher: do not rename.

## Still open
- frameworks/base (~48 lines): SystemUI `miku_dimens.xml`, `miku_styles.xml`, `ScrimController`, `NavigationBarInflaterView`, `quick_status_bar_expanded_header.xml`; core `miku_symbols.xml`, `miku_bools.xml`; `Instrumentation.java`, `AndroidKeyStoreSpi.java`, `package-shareduid-allowlist.xml`.
- Miku forks still pulled by snippets/miku.xml may contain Miku strings: ThemePicker, WallpaperPicker2, Themes, Gboard, vendor/settings, system/*, frameworks/{av,native,opt/telephony}. Grep a built image for "miku" to find real hits.
- Launcher3, build, soong licenses/branding not audited yet.
- Booted-device check of Settings > About phone (version/type/maintainer rows).
