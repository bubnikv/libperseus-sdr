# libperseus-sdr (SDR++ iak fork)

Driver library for the Microtelecom Perseus HF receiver, packaged with a
CMake build system for use as a third-party dependency by
[SDR++ iak](https://github.com/bubnikv/SDRPlusPlus).

## Fork chain

```
Microtelecom/libperseus-sdr  (autotools, last commit 2014, dormant)
        |
        v
AlexandreRouma/libperseus-sdr  (CMakeLists.txt overlay, sources moved to src/)
        |
        v
bubnikv/libperseus-sdr        (this fork)
```

* **Microtelecom** is the upstream maintained by Nicolangelo Palermo (IV3NWV),
  the original author of the Perseus driver. Build system: GNU autotools
  (`autoreconf -i && ./configure && make`).
* **AlexandreRouma** replaced the autotools layer with a single
  `CMakeLists.txt` so the library could be vendored alongside the rest of
  SDR++'s dependencies, and reorganized the source tree under `src/`.
  The C sources were lightly edited during that refactor.
* **bubnikv** (this repo) keeps Rouma's CMake build system but restores
  Microtelecom's correctness in the two places Rouma's refactor regressed.

## Differences vs Microtelecom upstream

| Area              | Microtelecom upstream                          | This fork                                                  |
| ----------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Build system      | GNU autotools (`configure.ac`, `Makefile.am`)  | CMake (`CMakeLists.txt`)                                   |
| Source layout     | Sources at repo root                           | Sources under `src/`                                       |
| `<unistd.h>`      | Included                                       | Included (restored — Rouma's fork had dropped it)          |
| Async stop loop   | n/a                                            | `Sleep(1)` on Windows, `usleep(1000)` on POSIX             |
| Distribution docs | `README`, `INSTALL`, `ChangeLog`, `Doxyfile`   | Dropped (build-system docs only)                           |
| `95-perseus.rules`| Shipped (udev rule)                            | Not shipped (vendored libs don't install system-wide rules)|

The Perseus protocol implementation, FX2 firmware uploader, FPGA bitstream
handling, and public API are identical to Microtelecom upstream — the
C/H files differ only in include order and the platform gate above.

## Differences vs AlexandreRouma/libperseus-sdr

* **`#include <unistd.h>`** restored in `src/perseus-sdr.c`. Rouma's
  refactor dropped it. `sleep(3)` then relied on an implicit declaration,
  which GCC 14+ (Debian trixie, Ubuntu noble, Fedora 40) promotes to a
  hard error (`-Wimplicit-function-declaration`).
* **`Sleep(1)`** in `perseus_stop_async_input` (a Rouma addition not in
  Microtelecom upstream) is now gated behind `_WIN32`. On POSIX the
  equivalent `usleep(1000)` is used. Without the gate, GCC 14+ rejects
  the implicit declaration of the Windows-only `Sleep`.
* **Variadic-macro trailing comma** in `src/perseus-sdr.h`'s `dbgprintf`
  and `errorset` macros: Rouma converted Microtelecom's GCC-style
  `args...` form to ISO `__VA_ARGS__`, but wrote `format, __VA_ARGS__`,
  which leaves a dangling comma when callers pass no variadic arguments
  (e.g. `errorset(PERSEUS_NOMEM, "can't alloc")`). Modern GCC rejects
  this as `expected expression before ')'`. Switched to the
  GCC/Clang/MSVC-supported `, ##__VA_ARGS__` form which swallows the
  comma when `__VA_ARGS__` is empty.

## Building

CMake project, no autotools:

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
cmake --install build --prefix /your/install/prefix
```

`libusb-1.0` is required. On Windows the build expects `LIBUSB_INCLUDE_DIRS`
and `LIBUSB_LIBRARIES` to be set by the caller (vcpkg/manifest-mode or
hand-fed by a parent build system). On Linux/macOS the build uses
`pkg-config` to find libusb unless `LIBUSB_FOUND` is pre-defined.

## License

LGPL-3.0-or-later, unchanged from Microtelecom upstream. See
`COPYING.LESSER`.
