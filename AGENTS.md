# AGENTS.md — NeoPod (custom firmware for the Fosi Audio DAP Neo)

Freestanding C11 firmware (Rockchip SoC, dual-core), adapted from EchoPod. Portable app in `src/`, SDL2 simulator in `platform/sim`, device drivers in `platform/device`, PC Python tools in `tools/`. Deep docs: `docs/architecture.md` (read this first), `docs/library-format.md`, `docs/hardware.md`, `docs/flash-neopod.md`.

## Build and test

```sh
cmake --preset dev && cmake --build --preset dev && ctest --preset dev
```

- `dev` preset = host build, Debug, UBSan on (`ECHO_SANITIZE=ON`). ASan is opt-in (`-DECHO_ADDRESS_SANITIZER=ON`) and hangs at startup on macOS 27 / Apple clang 17 — don't use it there.
- Single test: `ctest --preset dev -R '^test_power_policy$'` (test name = executable under `build/dev/tests/`).
- Python-tool tests (`tools_unit_tests`, `*_up_to_date`, `cover_decode_matches_pillow`) need a tools venv: `python3 -m venv .venv && .venv/bin/pip install -r tools/requirements.txt`. Without `.venv` they are silently skipped at configure time.
- Device cross-build (needs `arm-none-eabi-gcc`): `cmake --preset device && cmake --build --preset device` → `build/device/platform/device/echo_device.elf`; `tools/make_image.py` packs it into the `.IMG`. `release` preset = `device` + `ECHO_RELEASE=ON` (links the short release Diagnostics menu instead of the developer one).
- Production package (Windows, PowerShell): `python -m venv .venv-flasher`, install `tools/requirements-release.txt` into it, then `./build-neopod.ps1`. It builds with `-DECHO_BOARD=neo -DECHO_RELEASE=ON -DECHO_JACK_DIAGNOSTICS=OFF` into `build/neo` and writes `release/DP3_V182.IMG`, `release/sd-card/.echo/` companions, `SHA256SUMS.txt`, both flasher exes and the zip. The version `182` is hardcoded in the script (`DP3_V182.IMG`, `tools/build_neo_image.py --version 182`) — bump it there, in `README.md`, and in `release/` together. `ECHO_FIRMWARE_VERSION` lives in `platform/device/CMakeLists.txt`.
- No CI and no pre-commit hooks (`.github/workflows/` is empty). Enforcement is via ctest: `format` (clang-format), `no_comments`, `fonts_up_to_date`, `sort_fold_up_to_date`, `library_fixture_up_to_date`, `target_audit`. After editing C, run `clang-format -i` on the changed files (style in `.clang-format`); warnings are errors (`cmake/project_options.cmake`).

## Board selection

`ECHO_BOARD` = `echo-mini` (default, upstream) | `neo` (this firmware's device) | `neo-sim` (desktop-only; configuring with `-DECHO_BUILD_DEVICE=ON` and `neo-sim` is a fatal error). Host dev builds of the Neo UI use `neo`/`neo-sim` too (see `tests/CMakeLists.txt` `neo_render_ui`).

## Architecture in one paragraph

`src/` is layered and the CMake targets enforce it: `core` (no deps) ← `gfx`, `image`, `codec` ← `fonts`, `ui`, `library`, `playback`, `settings` ← `app` (composition root) ← `platform/*`, which implements `hal` and owns `main()`. Each module exposes only its own `include/<module>/`; anything else in the module dir is private. `platform/device/logic/` is hardware-free driver logic and is unit-tested on the host; `platform/device/board/` is device-only. On hardware, decoding runs on core 1 as companion binaries (`neo_flac/mp3/aac/opus.bin` in the image, mirrored to `sd-card/.echo/`), and the library rescan indexer is a CRC-checked overlay (`echo_rescan_overlay`, linked at `0x0307A000`).

## Hard code rules (all enforced by tests)

- **No comments in any C, CMake or Python file** (`no_comments` test, `tools/check_no_comments.py`). Prose goes in `docs/`. `third_party/` is exempt.
- No heap in `src/`; freestanding C11 including only `<stdint.h>`, `<stdbool.h>`, `<stddef.h>`. No recursion, no VLAs, device stack 16 KB. Static RAM over `ECHO_APP_RAM_BUDGET_BYTES` (128 KB) fails `target_audit` (Cortex-M3 cross-compile; skipped if no `arm-none-eabi-gcc`).
- Naming: types `PascalCase`, functions/variables `snake_case` with module prefix (`key_tracker_update`), constants `UPPER_SNAKE_CASE`, files `snake_case`. Functions do one thing, aim < 20 lines, max 4 params (group more into a struct).
- Device image limits enforced by the linker script: code < 171,028 bytes (`0x29C14`), `.bss` < `0x0304C000`, `.data` only the entry vector.

## Generated code — never edit by hand

- `src/fonts/` ← `tools/font_build.py` from `assets/fonts/`; `src/library/sort_fold_table.*` ← `tools/fold_table.py`. Regenerate with `.venv/bin/python tools/font_build.py --config assets/fonts/fonts.json --output src/fonts --style .clang-format` (and `tools/fold_table.py --output src/library --style .clang-format`). The `fonts_up_to_date` / `sort_fold_up_to_date` tests fail if committed output differs.

## Simulator

```sh
build/dev/platform/sim/echo_sim --card ~/Music --system-dir build/library   # windowed, SDL audio
build/dev/platform/sim/echo_sim --headless --frames 120 --keys down,down,play --screenshot shot.bmp
SDL_AUDIODRIVER=dummy build/dev/platform/sim/echo_sim ...                     # real-time path, no sound
```

`--card DIR` is read-only; `--system-dir DIR` holds `/.echo/` writes; writes outside it are refused. Headless mode is deterministic (virtual clock, 16 ms/frame).

## Safety

`flash-neo.py` / `flash_neo_gui.py` / `release/flash-neo.exe` write real firmware slots on hardware. Never run them against a connected device; recovery is maskrom only (`docs/device-recovery.md`). Card-image fixtures (`tests/fixtures/cards/`) are verified with macOS `fsck_msdos`/`fsck_exfat`, so the `test_fat` fixture checks are macOS-oriented.
