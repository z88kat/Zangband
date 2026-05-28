# Zangband

Zangband 2.7.5pre1 — a roguelike dungeon crawler in the Angband family, originally released in 2005 — built for **Apple Silicon Macs**.

This repository contains the 2005 source tree plus a hand-rolled build system that bypasses the original autoconf rig (which cannot produce a working build on modern macOS). Only the **ncurses terminal interface** is supported here. The original Carbon / Classic Mac / X11 / GTK / Tk ports are dead on ARM macOS and have not been revived.

## Requirements

- **Apple Silicon Mac** (M1 / M2 / M3 / M4). This build is not tested on Intel Macs.
- **macOS 13 Ventura or later** (anything that ships modern Apple Clang).
- **Xcode Command Line Tools**, which provide `clang`, `make`, and the macOS SDK (including `ncurses`):
  ```sh
  xcode-select --install
  ```

No Homebrew, no XQuartz, no third-party dependencies. The system `ncurses` is the only library linked.

## Build

```sh
cd src
make -f Makefile.macarm
```

This produces `./zangband` at the repo root. A clean rebuild takes about 10 seconds on an M-series Mac.

To start over from a clean state:

```sh
cd src
make -f Makefile.macarm clean
make -f Makefile.macarm
```

## Run

The game uses an ncurses TUI — run it from a real terminal (Terminal.app, iTerm2, etc.):

```sh
./zangband
```

Game data lives in `lib/`; the binary expects `./lib/` relative to the working directory, so run from the repo root.

To play with a specific savefile name:

```sh
./zangband -uSteven       # uses lib/save/<uid>.Steven
```

## How the build works

`src/Makefile.macarm` builds a single-port binary:

- Compiles all source files but enables only `USE_GCU` (ncurses). Every other `main-*.c` self-guards on its own `USE_xxx` macro and falls out to an empty object.
- Defines `SIZEOF_LONG_INT=8` so `h-config.h` enables `L64`, which makes `s32b` an `int` instead of a `long`. On ARM64 macOS `long` is 64 bits — without this, the RNG and savefile format silently break.
- Passes `-Wno-everything` to silence the strict warning fortress this code was tuned against in 2005, several of which modern Clang has promoted to hard errors.
- Passes `-fcommon` to restore pre-GCC-10 tentative-definition merging, which this codebase relies on (multiple `int foo;` declarations across translation units without `extern`).
- Builds the bundled `tolua` tool from `src/lua/`, then uses it to generate the eight `l-*.c` Lua-binding wrappers from `l-*.pkg`, then links the lot together with `-lncurses`.

No source files were edited. The fix is entirely in the build flags.

## What's *not* supported

- **GUI ports** — `main-mac.c` (Classic), `main-mac-carbon.c` / `main-crb.c` (Carbon — removed from macOS 10.15), `main-x11.c` / `main-xaw.c` (XQuartz), `main-gtk.c`, `main-win.c`, etc. These files still compile (to empty objects) but their `USE_xxx` macros are deliberately undefined.
- **Intel Macs.** This may work on x86_64 with the same makefile but has not been tested.
- **The original `configure` / `makefile` / `makefile.std`** at the repo root. They target a 2005 toolchain and Linux/X11 environment; on ARM macOS they fail in several places (autoconf cache, GTK config, Carbon frameworks, etc.). Ignore them.

## Repository layout

| Path | Purpose |
|---|---|
| `src/Makefile.macarm` | The ARM-macOS build (use this) |
| `src/*.c`, `src/*.h` | Game source |
| `src/lua/` | Bundled Lua 4.x interpreter + tolua binding generator |
| `src/l-*.pkg` | Lua binding definitions (compiled into `l-*.c` at build time) |
| `lib/` | Game data: monsters, items, help text, savefiles, etc. |
| `makefile`, `makefile.in`, `configure`, `configure.in` | Original 2005 autoconf build — ignore on ARM macOS |
| `src/makefile.std` | Original hand-rolled multi-platform makefile — ignore on ARM macOS |
| `readme`, `z_faq.txt`, `z_update.txt` | Upstream Zangband / Angband 2.8.1 documentation |
