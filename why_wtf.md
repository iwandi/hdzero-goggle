# Why WTF Report

## 1) Why the goggles do not support exFAT

The DVR/storage path is hard-wired to FAT32-style handling:

- SD format script always runs `mkfs.vfat -F 32` (`mkapp/app/script/formatsd.sh`).
- Record-side mount detection only treats filesystem type `0x00004d44` as mounted (`src/record/disk.c`), which is `MSDOS_SUPER_MAGIC` (FAT).
- No exFAT formatter or exFAT-specific mount/validation logic is present.

So exFAT is effectively unsupported by design in current code.

## 2) Why Low band is a separate option and not in the wheel channel menu

Channel wheel tuning uses a single channel index range and does not switch HDZero band:

- Wheel logic cycles `1..HDZERO_CHANNEL_NUM` only (`src/core/input_device.c`).
- `HDZERO_CHANNEL_NUM` depends on currently selected `g_setting.source.hdzero_band` (race vs low), so band must already be chosen.
- Band selection is exposed in Source settings as a dedicated toggle (`src/ui/page_source.c`).

So Low band is modeled as a mode-level source setting, not a tune-time wheel selection.

## 3) Why manual recording can fail even when SD card appears mounted

There are multiple "mounted" checks with different strictness:

- UI/runtime flag uses `sdcard_mounted()` from `src/util/sdcard.c` (mountpoint device-id check).
- Recorder-side status uses `disk_mounted()` from `src/record/disk.c`, and only returns mounted for FAT magic `0x00004d44`.
- If card is mounted but not FAT32 (or otherwise not writable), recorder can still fail start/open (`src/record/ffpack.c`).

Also, DVR start requires both `g_sdcard_enable` and `g_sdcard_ready` (`src/core/dvr.c`), so a card can look mounted while not yet "ready" for recording.

## 4) Why DVR container format changes even when a specific format is set (manual mode)

On every DVR start, firmware rewrites `record.type` from UI settings:

- `dvr_update_record_conf()` writes `"ts"` or `"mp4"` based on `g_setting.record.format_ts` (`src/core/dvr.c`).
- Record process reloads config at start (`conf_loadRecordParams`) (`src/record/main.c`).
- Parser only accepts `mp4`/`ts`; unsupported values fall back to default `REC_packTYPE` (`ts`) (`src/record/confparser.c`, `src/record/record_definitions.h`).

So manual mode affects start/stop behavior, but container choice is still enforced from firmware settings and parser rules.

## 5) Did backpack changes address any of these issues?

Partly, but only for ELRS/MSP channel handling:

- `cc50f3b` (`fix elrs lowband support`) added lowband-aware reporting/sending in `src/core/elrs.c`.
- `c96f1d2` (`elrs support E1 and F1 channel`) improved raceband channel coverage.
- `480bc61` (`Fix ELRS MSP channel and frequency handlings`) refactored ELRS MSP channel/frequency logic.
- `2b6761b` (`Fix ELRS baclpack DVR timed start/stop logic`) only addressed timed DVR commands from the backpack.

These commits do **not** add exFAT support, do **not** unify Low band into the wheel menu, and do **not** change the DVR config rewrite behavior described above.

## 6) Why ELRS Backpack still cannot switch to Lowband when requested

Current ELRS receive-side logic can describe/send lowband, but it still cannot switch the goggles into lowband from an incoming backpack request:

- `MSP_SET_BAND_CHAN` converts backpack index to a plain HDZero channel through `hdz_index2ch()` (`src/core/elrs.c`).
- `hdzero_channel_map` leaves all Lowband entries as `0`, so incoming L1-L8 indices do not map to valid HDZero channels.
- `channel_channel_hdzero()` only sets `g_setting.scan.channel`; it does **not** update `g_setting.source.hdzero_band`.
- `MSP_SET_FREQ` has the same limitation: lowband frequencies resolve to indices whose reverse map is `0`, so they are rejected.

So the backpack can work with Lowband only if the goggles are **already** in Lowband. It cannot switch the band by itself because the incoming MSP path has no valid lowband reverse mapping and no band-change step.
