# Changelog

All notable changes to TamaPoke are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Version
numbers match `FW_VERSION` in `TamaPoke.ino`, shown on the settings screen and printed
over serial at boot.

## [Unreleased]

### Added

- CI on GitHub Actions (`.github/workflows/ci.yml`), three jobs: compile the firmware
  against the exact FQBN and libraries the README documents (pinned to
  `esp32:esp32@3.3.11`, the version verified to build cleanly, with `--warnings=all`);
  regenerate `include/dex.hpp` and `include/species.hpp` and fail if that produces a
  diff, so a hand-edited generated file gets caught; and lint `tools/` with `ruff`.
  The lint job is scoped to pyflakes correctness rules only (`ruff.toml`) — the
  scripts predate any style convention, and a full style pass is separate, larger
  work than wiring up CI.

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
- `Pet::level()` no longer wraps back to 0 at ~10.6 days of continuous play
  (`uint8_t` overflow at level 256, with `MINUTES_PER_LEVEL=60`). The game itself
  invites reaching that point: "stay together" can postpone a farewell
  indefinitely. `level()` and `evoDeclinedLv` are now `uint16_t` (cosmetically
  capped at 999, never wrapped), and `canEvolveNow()` no longer truncates
  `evolveLevel + careMistakes` back to `uint8_t` before comparing against it.
- The `HEALTH` heartbeat and serial command now report `ESP.getFreePsram()` and
  `heap_caps_get_largest_free_block(MALLOC_CAP_SPIRAM)` alongside the existing heap
  numbers. Sprites and the framebuffer live in PSRAM, not heap, so `heap`/`min` alone
  couldn't predict a `ps_malloc()` failure from PSRAM fragmentation — exactly the
  metric the pending 24-48h soak test needs.
- The `PUT` protocol (SD provisioning over USB) now checks that `f.write()` actually
  wrote every byte instead of ignoring its return value — previously a full or failing
  card would still get a `DONE`, leaving a silently truncated file on the SD (exactly
  the kind of corrupt input the earlier TPK2/`thumbs.bin` fixes above had to guard
  against). It also rejects any destination path that doesn't resolve under `/mons/`
  or that contains `..`, instead of writing wherever the line said to.
- `handleSerial()` no longer blocks the game loop for up to a second on stray serial
  bytes with no trailing newline (`Serial.readStringUntil('\n')`'s timeout). It now
  assembles lines byte-by-byte from `Serial.available()` and only dispatches once a
  full line has arrived, in a new `processSerialLine()`.
- `galleryTap()` and `keyboardTap()` now reject touches above/left of their grid
  before dividing, instead of after. Integer division truncates toward zero, not
  toward -∞, so a touch just outside the grid's top-left edge produced column/row 0
  instead of a negative value the existing `< 0` guard could catch — on this round
  screen, the edges are exactly where a finger rests without meaning to. Concretely:
  tapping the round screen's left edge used to open Pokédex entry #1's detail view,
  and tapping just above the keyboard used to type the letter 'A'.
- Synced the README's firmware badge (was `v1.2`, `FW_VERSION` is `1.4`) and fixed
  three comments in `include/pet.hpp`/`src/pet.cpp` that described the farewell
  ceremony as "final form + 7 days" — `FAREWELL_AGE_MIN` is 3 days, and the README
  already documented 3 correctly. Also fixed `web/index.html`'s sprite-bundle-size
  log message, still saying "~58 MB" after the README and `web/README.md` were
  already corrected to the real ~40 MB.
- The Pokédex gallery's thumbnails now reload after receiving files over USB.
  `ensureMon()` already reloaded the active PMD sprite when `sdDirty` was set after a
  `PUT`, but never touched `thumbs` (loaded once in `setup()`), so `thumbs.bin` sent
  by the web installer stayed invisible until the next reboot.
- `web/index.html`'s `esp-web-tools` script tag now pins an exact version (`10.4.0`
  instead of the floating `@10` range) and carries Subresource Integrity, so a
  changed or compromised CDN file fails closed instead of running silently. (SRI only
  covers this entry file, not the chunks it dynamically `import()`s afterward —
  closing that fully would mean vendoring the whole `dist/web/` chunk graph, more
  than this fix takes on.)
- The sprite upload (`sendAll()`) now survives a dropped connection or reload:
  successfully confirmed files are remembered in `localStorage` (by name + size) and
  skipped on a retry, instead of resending the full ~40 MB / ~10 min transfer from
  scratch. The remembered set is cleared once every file in a run is confirmed.

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
