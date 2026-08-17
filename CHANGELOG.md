# Changelog

All notable changes to TamaPoke are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Version
numbers match `FW_VERSION` in `TamaPoke.ino`, shown on the settings screen and printed
over serial at boot.

## [Unreleased]

### Changed

- Reorganized the sketch's own headers and sources into `include/` and `src/`, following
  Arduino's `src` sketch convention (compiled recursively, not shown as IDE tabs).
  `pin_config.h`, `dex.h`, `species.h`, `pet.h`, `audio.h`, `rtcbat.h`, `sdmon.h` and
  `i18n.h` are now `include/*.hpp`; their `.cpp` counterparts moved to `src/*.cpp`.
  `TamaPoke.ino` stays at the sketch root, as required by the Arduino build.
- Updated `tools/gen_dex.py` and `tools/sprites.py` to write their generated headers to
  `include/dex.hpp` and `include/species.hpp`.

### Removed

- Removed the TPK1 legacy sprite fallback (`SdMon` struct, `drawPetSD()`, and its call
  sites in `ensureMon()`, `renderGame()`, `drawPet()` and the `HEALTH`/`STATS` serial
  commands): no packer in the repo has produced that format since the project moved to
  PMD/TPK2, so the path was unreachable in practice.
- Removed `Pet::feed()`, an unused compatibility wrapper with no remaining callers
  (`feedBerry()`/`feedCandy()` are used instead).
- Removed unused fields from the generated `Species` struct (`name`, `type`,
  `evolvesTo`, `evolveLevel`, `accent`), the `ElementType` enum, `SPRITE_W`, and
  `STARTERS`/`NUM_STARTERS` — all dead since `dex.h`/`DEX_TBL` took over species
  identity and evolution data. Regenerated `include/species.hpp` via
  `tools/sprites.py emit`; sprite pixel data is unchanged.
- Removed `tools/import_gif.py`, an orphaned importer the project's own docs already
  described as obsolete.
- Stopped tracking the generated sprite and firmware binaries in git
  (`tools/sdcard/mons/*.bin`, `web/sprites.pak`, `web/firmware/tamapoke.bin`): ~80 MB of
  regenerable binary blobs don't belong in version control. Added them to
  `.gitignore`; the README and `web/README.md` now make clear these must be built
  locally (`tools/pack_pmd.py` + `tools/make_thumbs.py`, or `tools/build_web.sh` for
  the web bundle) before flashing or deploying — they were never meant to ship in the
  repo itself.

### Fixed

- `PmdMon::load()` now rejects zero-length frame durations in TPK2 sprites (clamped to
  100 ms), and `pmdFrameAt()` is bounded by a frame-count guard regardless. A sprite
  file with every duration set to 0 used to spin `pmdFrameAt()` forever inside the
  render loop — a hang triggerable by any corrupt or malicious `.bin` reaching the SD
  card over USB.
- `SdThumbs::load()` now validates `thumbs.bin`'s size before allocating, and checks
  that the offset table and every thumbnail blob it points to actually fit inside the
  file before `SdThumbs::get()` hands out a pointer. Previously a truncated or corrupt
  `thumbs.bin` (a real risk: the web installer's transfer takes minutes) could read
  past the PSRAM allocation — garbage on screen at best, a crash at worst. Added
  `SdThumbs::unload()`, which the struct never had.

## [1.4] - 2026-08-07

### Fixed

- Bond was being lost about 12x faster than it could be gained from care, cooling much
  more than a day of good care could offset. Rebalanced the care-mistake bond penalty.

## [1.3] - 2026-08-07

### Fixed

- The egg could hatch on its own while the player was still choosing a starter on first
  run, skipping the starter choice.

## Earlier versions

Firmware releases before 1.3 (including the initial 1.0–1.2 line) predate this
changelog; see the git history for details.
