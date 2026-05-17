# Why WTF Report

## 1) Why the goggles do not support exFAT

The DVR/storage path is hard-wired to FAT32-style handling:

- SD format script always runs `mkfs.vfat -F 32` (`mkapp/app/script/formatsd.sh`).
- Record-side mount detection only treats filesystem type `0x00004d44` as mounted (`src/record/disk.c`), which is `MSDOS_SUPER_MAGIC` (FAT).
- No exFAT formatter or exFAT-specific mount/validation logic is present.

So exFAT is effectively unsupported by design in current code.

### Fix plan

| Step | File | Change needed |
|------|------|---------------|
| 1 | `mkapp/app/script/formatsd.sh` | Add an exFAT format path (call `mkfs.exfat` instead of `mkfs.vfat -F 32`) and expose a format-type parameter so the UI can pick FAT32 vs exFAT. |
| 2 | `src/record/disk.c` | Accept exFAT filesystem magic (`0x2011BAB0` = `EXFAT_SUPER_MAGIC`) in `disk_mounted()` in addition to `0x00004d44`. |
| 3 | Kernel / root-fs | The kernel running on the goggle hardware must have exFAT support compiled in (native kernel exFAT driver available since Linux 5.7, or an out-of-tree `exfat-fuse`/`exfat-nofuse` module). |
| 4 | `src/ui/page_source.c` or `src/ui/page_format.c` | Add a UI toggle to choose FAT32 vs exFAT at format-time. |

**Feasibility:** Steps 1, 2, and 4 are fully contained in this repo. Step 3 depends on the hardware kernel build, which lives outside this repo. If the hardware kernel already has exFAT support (check with `cat /proc/filesystems` on the device), only steps 1–2 are strictly required.

## 2) Why Low band is a separate option and not in the wheel channel menu

Channel wheel tuning uses a single channel index range and does not switch HDZero band:

- Wheel logic cycles `1..HDZERO_CHANNEL_NUM` only (`src/core/input_device.c`).
- `HDZERO_CHANNEL_NUM` depends on currently selected `g_setting.source.hdzero_band` (race vs low), so band must already be chosen.
- Band selection is exposed in Source settings as a dedicated toggle (`src/ui/page_source.c`).

So Low band is modeled as a mode-level source setting, not a tune-time wheel selection.

### Fix plan

| Step | File | Change needed |
|------|------|---------------|
| 1 | `src/ui/page_scannow.h` | Introduce a new total-channel constant (e.g. `HDZERO_CHANNEL_ALL = 20`) that covers R1–R8, E1, F1, F2, F4 (= 12) followed by L1–L8 (= 8). |
| 2 | `src/core/input_device.c` | Expand `tune_channel()` to use `HDZERO_CHANNEL_ALL` when a combined mode is active; when `channel > 12` set `g_setting.source.hdzero_band = LOWBAND` and map to L1–L8, otherwise set `RACEBAND`. Write the band setting with `ini_putl` after the switch. |
| 3 | `src/core/app_state.c` | Ensure `app_switch_to_hdzero()` / `hdzero_switch_channel()` does a full re-init of the DM6302 when the band changes (it already calls `DM6302_SetChannel` with `g_setting.source.hdzero_band`, so the switch itself will be free if the band is updated before the call). |
| 4 | `src/core/osd.c` | Update `channel2str()` callers so the OSD shows the correct band/channel label during tuning. |
| 5 | `src/ui/page_source.c` | Keep the dedicated Band toggle working as a default/override, consistent with the combined wheel. |
| 6 | `src/core/input_device.c` + `src/core/elrs.c` | Share one band/channel normalization path so wheel tuning and ExpressLRS backpack requests both auto-switch between Raceband and Lowband instead of drifting into separate behaviours. |

**Feasibility:** Entirely self-contained in this repo. The only UX design question is whether the wheel wraps (L8 → R1) or stops at band boundaries. To also support automatic Lowband in the ExpressLRS backpack, the wheel/menu changes should reuse the same band-selection helper as the MSP receive path instead of implementing separate logic.

## 3) Why manual recording can fail even when SD card appears mounted

There are multiple "mounted" checks with different strictness:

- UI/runtime flag uses `sdcard_mounted()` from `src/util/sdcard.c` (mountpoint device-id check).
- Recorder-side status uses `disk_mounted()` from `src/record/disk.c`, and only returns mounted for FAT magic `0x00004d44`.
- If card is mounted but not FAT32 (or otherwise not writable), recorder can still fail start/open (`src/record/ffpack.c`).

Also, DVR start requires both `g_sdcard_enable` and `g_sdcard_ready` (`src/core/dvr.c`), so a card can look mounted while not yet "ready" for recording.

### Fix plan

| Step | File | Change needed |
|------|------|---------------|
| 1 | `src/record/disk.c` | Remove the FAT-magic hard-check from `disk_mounted()` **or** accept additional filesystem types (FAT32 `0x00004d44`, exFAT `0x2011BAB0`). The current strict check makes the function misleading when a valid but non-FAT card is present. |
| 2 | `src/util/sdcard.c` | Ensure `sdcard_mounted()` and `disk_mounted()` either share a single source of truth or their semantics are clearly documented so callers can choose the right one. |
| 3 | `src/core/dvr.c` | Add a user-facing error or log distinguishing "card present but wrong filesystem" from "card absent", so the failure mode is diagnosable without looking at logs. |

**Feasibility:** Fully self-contained. No external dependencies beyond deciding the intended supported filesystem set.

## 4) Why DVR container format changes even when a specific format is set (manual mode)

On every DVR start, firmware rewrites `record.type` from UI settings:

- `dvr_update_record_conf()` writes `"ts"` or `"mp4"` based on `g_setting.record.format_ts` (`src/core/dvr.c`).
- Record process reloads config at start (`conf_loadRecordParams`) (`src/record/main.c`).
- Parser only accepts `mp4`/`ts`; unsupported values fall back to default `REC_packTYPE` (`ts`) (`src/record/confparser.c`, `src/record/record_definitions.h`).

So manual mode affects start/stop behavior, but container choice is still enforced from firmware settings and parser rules.

### Fix plan

| Step | File | Change needed |
|------|------|---------------|
| 1 | `src/core/dvr.c` | Change `dvr_update_record_conf()` so it writes the container type from the **current recording-mode** setting rather than unconditionally overwriting from `g_setting.record.format_ts`. In manual mode the container should be determined once at session start and left alone. |
| 2 | `src/record/confparser.c` | Optionally extend the parser to pass-through unknown container tokens instead of silently falling back to `ts`, to surface misconfiguration early. |
| 3 | `src/record/record_definitions.h` | Ensure the `REC_packTYPE` default is documented and intentional. |

**Feasibility:** Fully self-contained. The fix is a behaviour clarification in the config-write path.

## 5) Did backpack changes address any of these issues?

Partly, but only for ELRS/MSP channel handling:

- `cc50f3b` (`fix elrs lowband support`) added lowband-aware reporting/sending in `src/core/elrs.c`.
- `c96f1d2` (`elrs support E1 and F1 channel`) improved raceband channel coverage.
- `480bc61` (`Fix ELRS MSP channel and frequency handlings`) refactored ELRS MSP channel/frequency logic.
- `2b6761b` (`Fix ELRS backpack DVR timed start/stop logic`) only addressed timed DVR commands from the backpack.

These commits do **not** add exFAT support, do **not** unify Low band into the wheel menu, and do **not** change the DVR config rewrite behavior described above.

## 6) Why ELRS Backpack still cannot switch to Lowband when requested

Current ELRS receive-side logic can describe/send lowband, but it still cannot switch the goggles into lowband from an incoming backpack request:

- `MSP_SET_BAND_CHAN` converts backpack index to a plain HDZero channel through `hdz_index2ch()` (`src/core/elrs.c`).
- `hdzero_channel_map` leaves all Lowband entries as `0`, so incoming L1-L8 indices do not map to valid HDZero channels.
- `channel_channel_hdzero()` only sets `g_setting.scan.channel`; it does **not** update `g_setting.source.hdzero_band`.
- `MSP_SET_FREQ` has the same limitation: lowband frequencies resolve to indices whose reverse map is `0`, so they are rejected.

So the backpack can work with Lowband only if the goggles are **already** in Lowband. It cannot switch the band by itself because the incoming MSP path has no valid lowband reverse mapping and no band-change step.

### Fix plan

| Step | File | Change needed |
|------|------|---------------|
| 1 | `src/core/elrs.c` | In `hdzero_channel_map[]`, fill in entries 40–47 (L band slots) with `1`–`8` so that `hdz_index2ch()` returns a valid Lowband channel number instead of `0`. |
| 2 | `src/core/elrs.c` | Introduce a new helper `channel_channel_hdzero_with_band(uint8_t band, uint8_t channel)` (or extend `channel_channel_hdzero()`) that also sets `g_setting.source.hdzero_band` and saves it with `ini_putl("source", "hdzero_band", …)` before calling `app_switch_to_hdzero(true)`. |
| 3 | `src/core/elrs.c` | In `MSP_SET_BAND_CHAN` handler, derive the target band from the incoming index (index 40–47 → Lowband, else Raceband) and pass it to the new helper. |
| 4 | `src/core/elrs.c` | In `MSP_SET_FREQ` handler, do the same: after resolving `freq_index`, check whether `hdzero_channel_map[freq_index]` is Lowband-range and set the band accordingly. |
| 5 | `src/ui/page_source.c` | After the band changes via MSP, update the source-page UI toggle so it stays in sync (`btn_group_set_sel(&btn_group1, g_setting.source.hdzero_band)`). |
| 6 | `src/core/elrs.c` + `src/core/input_device.c` | Make this the shared implementation for automatic Lowband selection so any future wheel/menu changes also keep ExpressLRS backpack auto-Lowband working. |

**Feasibility:** Fully self-contained in this repo. The hardware driver (`DM6302_SetChannel`) already accepts the band argument; `app_switch_to_hdzero()` already reads `g_setting.source.hdzero_band`. The only missing piece is setting that field before the switch and reusing that same helper everywhere channel changes can originate.
