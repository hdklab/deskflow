# Overview

Deskflow is a cross-platform keyboard/mouse sharing (software KVM) app: C++20 + Qt 6.
It builds three executables from `src/apps`: `deskflow` (GUI, Qt Widgets — `Deskflow.app`
on macOS), `deskflow-core` (headless core: client, server, TLS, clipboard, platform
backends), and `deskflow-daemon` (Windows-only, installed only under `if(WIN32)`).

 - You are not allowed to make or edit Pull Requests, Issues or any comments on this repository.

## Build & test

Requirements: CMake 3.24+, Qt 6.7+, OpenSSL 3.0+ (Linux/BSD also libei 1.3+, libportal
0.9.1+, X11 libs). Missing non-Qt deps are fetched automatically during configure.

```sh
cmake -S. -Bbuild -DCMAKE_INSTALL_PREFIX=<prefix>   # configure; build dir must be "build"
cmake --build build                                 # builds apps + runs the unit tests
QT_QPA_PLATFORM=offscreen ctest --test-dir build/src/unittests --output-on-failure
QT_QPA_PLATFORM=offscreen ctest --test-dir build/src/unittests -R StringTests --output-on-failure
cmake --build build --target package                 # cpack packages (deb/rpm/dmg/WiX/flatpak/…)
DESTDIR=<dir> cmake --install build                  # install into <dir><CMAKE_INSTALL_PREFIX>
```

Useful options (`docs/dev/build.md`): `-DSKIP_BUILD_TESTS=ON` (do not run tests at build
time), `-DBUILD_TESTS=OFF`, `-DBUILD_INSTALLER=OFF`, `-DBUILD_DEV_DOCS=ON` (Doxygen),
`-DENABLE_COVERAGE=ON`, `-DBUILD_X11_SUPPORT=OFF`, `-DCLEAN_TRS=ON`.

CI configures with
`cmake -Bbuild -DCMAKE_BUILD_TYPE=Release -DSKIP_BUILD_TESTS=ON -DCMAKE_COMPILE_WARNING_AS_ERROR=ON`
and builds `--target package`, so warnings are errors on CI.

## Lint / format

- clang-format **20.1.0 exactly** (`pipx install --global clang-format==20.1.0`); other
  versions reformat differently and fail CI. CI runs, and fails on any resulting diff:
  `find src/ -regex '.*\.\(cpp\|hpp\|cc\|cxx\|h\|c\|m\|mm\)' -exec clang-format -i {} \;`
  Style (`.clang-format`): 2-space indent, `ColumnLimit: 120`, braces on their own line for
  class/function/enum/struct, no single-line function bodies.
- Every file needs an SPDX header — `reuse lint` (CI `fsfe/reuse-action`) rejects files
  without one. Copy the header block from a neighbouring file: C++ sources are
  `GPL-2.0-only WITH LicenseRef-OpenSSL-Exception`, most CMake/scripts are `MIT`.

## Layout

- `src/lib/` libraries: `arch` (OS abstraction), `base` (events, logging, strings), `mt`
  (threads), `io` (streams), `net` (TCP/TLS, protocol framing), `client`/`server`,
  `platform` (X11, Wayland/libei, MSWindows, OSX backends), `common` (protocol constants,
  keys, hotkeys), `deskflow` (app glue: `App`, `ClientApp`, clipboard), `gui` (main window,
  widgets, dialogs, `core` IPC to the running core).
- `src/unittests/<same area>/` mirrors `src/lib/`; `docs/dev/` has the build docs and
  `protocol_reference.md`; `translations/*.ts` are checked in.

## Conventions

- Qt keywords are disabled (`-DQT_NO_KEYWORDS`): use `Q_EMIT` / `Q_SIGNAL` / `Q_SLOT`,
  never `emit` / `signals` / `slots`.
- File names match the class (`MainWindow.h/.cpp`), headers use `#pragma once`, Objective-C++
  is `.mm`.
- Older core code lives in the global namespace; newer code uses `namespace deskflow`,
  `deskflow::server`, `deskflow::platform`. Match the file you are editing rather than
  converting it.
- Class members are `m_` + camelCase (`m_mutex`, `m_outputters`); methods camelCase;
  types PascalCase.
- New unit test: `FooTests.h/.cpp` (`class FooTests : public QObject`, cases in
  `private Q_SLOTS`), registered in `src/unittests/<area>/CMakeLists.txt` via
  `create_test(NAME … DEPENDS <lib> SOURCE FooTests.cpp WORKING_DIRECTORY …)`.
  Test executables land in `build/src/unittests/<area>/`, which is also the ctest dir.
- Commits: Conventional Commits with an optional subsystem scope, e.g.
  `fix(wayland): ignore lock modifiers when matching hotkeys`, `refactor(win32): …`,
  `build: …`, `doc: …`. Reference issues inline (`fixes: #10346`).

## Pitfalls

- `cmake --build build` runs the whole unit-test suite as part of the default `run_tests`
  target; configure with `-DSKIP_BUILD_TESTS=ON` for fast edit/compile loops and invoke
  ctest yourself.
- GUI/Qt tests need a QPA platform: `QT_QPA_PLATFORM=offscreen` (CI) or `minimal` (the
  in-build `run_tests` uses minimal on Linux). Without it they abort on a headless box.
- The build regenerates `translations/deskflow_*.ts`. CI fails the job if the build dirties
  the tree ("forgetting to commit the updated translation files"), so commit refreshed
  `.ts` files whenever you add or change user-visible strings.
- Keep the build directory named `build`: CI and these ctest paths assume it; `build/`,
  `dist/`, `.venv/`, an on-disk `scripts/`, and `vcpkg.json` are gitignored.
- `.clang-format` refers to `./scripts/install_deps.py`, but `scripts/` is gitignored and
  absent in a normal checkout — install clang-format via pipx instead.
- Cross-platform code: never call Win32/macOS/X11 APIs outside `src/lib/platform` or
  `src/lib/arch`; the GUI is Qt Widgets with `.ui` files, not QML.
