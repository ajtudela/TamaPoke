# Changelog

All notable changes to TamaPoke are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Version
numbers match `FW_VERSION` in `TamaPoke.ino`, shown on the settings screen and printed
over serial at boot.

## [Unreleased]

### Added

- CI on GitHub Actions (`.github/workflows/ci.yml`), four jobs: compile the firmware
  against the exact FQBN and libraries the README documents (pinned to
  `esp32:esp32@3.3.11`, the version verified to build cleanly, with `--warnings=all`);
  regenerate `include/dex.hpp` and `include/species.hpp` and fail if that produces a
  diff, so a hand-edited generated file gets caught; lint `tools/` with `ruff`
  (scoped to pyflakes correctness rules only, `ruff.toml` — the scripts predate any
  style convention, and a full style pass is separate, larger work than wiring up
  CI); and the i18n format-specifier consistency check below.
- `tools/test_i18n_formats.py`: parses `STRINGS[][]` and the three `MED_*[][]` medal
  tables straight out of `src/i18n.cpp` and checks, for every string ID, that all 6
  languages use the same `%`-format specifiers in the same order — several of these
  strings are used as `snprintf()` format arguments (not literals), so the compiler
  can't catch a mismatched translation with `-Wformat`; a wrong specifier is
  undefined behavior that only shows up in the affected language. Also checks every
  language row has exactly as many entries as `StrId`/`MED_COUNT` expect: `STRINGS`
  is declared with both array dimensions fixed, so a row with too *few* strings
  doesn't fail to compile — the missing slots are silently null-initialized instead.
  Verified to both pass on the current tables and correctly fail when a specifier
  was deliberately removed from one language's string, restored before committing.

### Changed

- Merged `drawScene()` (main screen) and `drawGameScene()` (minigame/training-bag
  background) into one shared `drawBackground()`. Both drew an identical hour-of-day
  sky gradient and biome ground color; only `drawScene()` also drew the sun/moon/
  clouds and the biome-specific ground details (beach sea, horizon hill, forest/
  volcano/mountain/snow/meadow silhouettes) — those stay exactly as before, gated by
  new `sunMoonClouds`/`details` parameters, so neither screen's look changes. Also
  added `isSceneNight()` so the night check (`sceneHour() < 6 || sceneHour() >= 20`)
  is computed once per render instead of calling `sceneHour()` twice per evaluation
  (short-circuit `||` re-evaluates it when the first half is false) and separately
  again in each of `renderSack()`/`renderGame()`/the old `drawGameScene()`.
- Added `blitIndexed()`, a shared indexed-bitmap blitter that groups same-index
  horizontal runs into one `fillRect()` instead of drawing one per source pixel —
  sprites have large flat regions, so this typically cuts the number of draw calls
  3-6x. Rewired `drawThumb()` (gallery thumbnails) and `drawPmdActM()` (the PMD
  creature animation, the two highest-volume blit sites — up to ~2300 calls per
  frame for a 48×48 sprite at scale 5) to use it; `drawMap()` (the small flash-
  fallback sprites and UI icons) was left as-is, since its character-keyed pixel
  format (`spriteColor(ch)`) isn't index-based like the other two, and unifying it
  would mean forcing an artificial shared representation onto a much lower-volume,
  differently-shaped code path for little real gain.

### Fixed

- The ball minigame's physics no longer run at "one step per rendered frame".
  `stepGame()` treated gravity/velocity/the chasing pet's speed as fixed per-call
  increments, but calls happen whenever `loop()`'s frame scheduler decides to render
  — which takes longer for a bigger sprite — so the ball fell measurably slower with
  a large species loaded (e.g. Charizard) than a small one (e.g. Diglett), and the
  minigame's high score wasn't comparable across species. `stepGame()` now measures
  real elapsed time since its last call and scales gravity, position, and the pet's
  chase speed by that (relative to the ~85 ms step the constants were originally
  tuned around, capped at 3x to avoid a large jump after an unusually late frame).

### Changed

- `Pet::save()`/`Pet::load()` now use a single versioned NVS blob (`PetSaveV2`, one
  `putBytes("save", ...)`/`getBytes(...)` call) instead of 35 separate NVS keys.
  Every player action called `save()`, so this was 35 individual flash writes per
  action instead of one; it also wasn't atomic — a power cut mid-`save()` could leave
  a mixed old/new state across those 35 keys. Saves from before this change are
  migrated automatically and losslessly: `load()` tries the new blob first, and only
  if it's absent (or fails a magic/version sanity check) falls back to
  `loadLegacyV1()` — the old 35-key reading logic, unchanged and kept around
  indefinitely, including its own nested sub-migration from the pre-Pokédex-numbers
  save format — then immediately writes the new blob so the *next* save is already
  the fast, atomic path. The old keys are deliberately left in NVS rather than
  deleted (harmless, and a safety margin — a rollback to older firmware still finds
  its data).
  - `syncClock()` read the previous session's timestamp via a direct
    `prefs.getUInt("seen", 0)`, bypassing the object's own state; that key stops
    being written once a player is on the new blob, which would have silently broken
    offline-progression calculation. Changed to read `lastSeenEpoch` (already loaded
    by `load()`) instead — behaves identically today, and keeps working after
    migration.
  - Preserved one exact pre-existing behavior that a single-blob write could easily
    have dropped: a `save()` while `lastSeenEpoch` is momentarily 0 (RTC reporting no
    valid time) must not erase the last known-good timestamp. Added `savedSeenEpoch`,
    updated only when `lastSeenEpoch` is non-zero, and persisted in its place.
  - `checkMedals()` (reached from `tick()`, the once-a-minute callback) called
    `save()` directly whenever a medal was earned, defeating the deferred-save
    mechanism `tick()` itself otherwise respects (a synchronous NVS write freezes
    both cores for about a second — exactly what deferring saves until the screen
    dims exists to avoid). Now sets `pendingSave = true` instead. `hatch()` and
    `registerCare()`, the other two callers, already call `save()` themselves right
    after `checkMedals()`, so this doesn't change their behavior (removes one
    redundant write there, actually).

  Verified the new blob format's on-the-wire layout with a standalone native test
  (not part of the build, compiled with host g++): filled every field of
  `PetSaveV2` with boundary-leaning values (min/max integers, a 12-character nick,
  non-trivial `dexReg`/`dexShinyReg` bit patterns), round-tripped the struct through
  a raw byte buffer exactly as `putBytes()`/`getBytes()` would, and confirmed every
  field matches — `sizeof(PetSaveV2)` is 105 bytes, comfortably within NVS blob
  limits. Also confirmed a garbage magic or wrong version is correctly rejected.

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
