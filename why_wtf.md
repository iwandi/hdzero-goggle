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
