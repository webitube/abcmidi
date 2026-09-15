# abcmidi — Build & DevOps Guide

This document describes how to build the **abcmidi** project (ABC music notation
tools: converters between ABC, MIDI, and PostScript), how to run its test suite,
and the specific source/build changes that were required to make the project
build cleanly on **Windows (MSVC)**.

---

## 1. Overview

abcmidi is a C project that produces eight command-line tools:

| Binary      | Purpose                                            |
| ----------- | -------------------------------------------------- |
| `abc2midi`  | ABC notation → MIDI                                |
| `abc2abc`   | ABC → ABC (transposition, reformatting)            |
| `midi2abc`  | MIDI → ABC notation                                |
| `midistats` | MIDI file statistics                               |
| `mftext`    | MIDI → human-readable text                         |
| `yaps`      | ABC → PostScript                                   |
| `midicopy`  | MIDI file filtering / extraction                   |
| `abcmatch`  | ABC tune matching / comparison                     |

The project supports two build systems:

- **CMake** (primary, cross-platform) — `CMakeLists.txt` + `CMakePresets.json`
- **Autotools / Make** (legacy, Unix) — `configure`, `configure.ac`, `Makefile.in`

The CMake build is the recommended path and is what this guide focuses on.

---

## 2. Prerequisites

| Tool            | Minimum version | Notes                                                        |
| --------------- | --------------- | ------------------------------------------------------------ |
| CMake           | 3.14            | 3.21+ recommended (required for `CMakePresets.json` v3)      |
| C compiler      | any modern      | MSVC (Visual Studio 2022), GCC, or Clang                     |
| CTest           | (bundled)       | Ships with CMake; used by the test suite                      |

No external libraries are required. On non-Windows platforms the standard math
library (`libm`) is linked; on Windows the math functions are part of the CRT,
so no extra library is needed.

---

## 3. Building with CMake

The project ships `CMakePresets.json` with three configure presets:

| Preset     | Build type | Notes                                        |
| ---------- | ---------- | -------------------------------------------- |
| `default`  | Release    | Standard optimized build (recommended)       |
| `debug`    | Debug      | Debug symbols, no optimization               |
| `sanitize` | Debug      | Debug + AddressSanitizer + UBSan (GCC/Clang) |

### 3.1 Configure

```sh
# Release (default)
cmake --preset default

# or Debug
cmake --preset debug

# or Debug + sanitizers (GCC/Clang only)
cmake --preset sanitize
```

Each preset writes build files into `build/<presetName>/` (e.g.
`build/default/`). Use `--fresh` to wipe and reconfigure from scratch:

```sh
cmake --preset default --fresh
```

### 3.2 Build

```sh
# Release
cmake --build build/default --config Release

# Debug
cmake --build build/debug --config Debug
```

> **Multi-config generators** (Visual Studio, Xcode) ignore `CMAKE_BUILD_TYPE`
> and use `--config` to select the configuration. **Single-config generators**
> (Makefiles, Ninja) bake the build type in at configure time, so `--config`
> is optional there.

### 3.3 Build outputs

With the Visual Studio generator, executables land in
`build/default/Release/`:

```
build/default/Release/
├── abc2abc.exe
├── abc2midi.exe
├── abcmatch.exe
├── mftext.exe
├── midi2abc.exe
├── midicopy.exe
├── midistats.exe
└── yaps.exe
```

With a single-config generator (e.g. Ninja/Makefiles) the binaries are placed
directly in `build/default/`.

### 3.4 Install (optional)

```sh
cmake --install build/default --prefix /usr/local
```

Installs the eight binaries to `<prefix>/bin`, plus the top-level documentation
files and man pages.

---

## 4. Testing

The test suite (defined in `tests/CMakeLists.txt`) has two layers:

1. **Smoke tests** — every binary is run with `-ver` and must exit cleanly.
   Catches link errors, missing libraries, and trivial crashes.
2. **Golden tests** — each program is run on a sample input from `samples/`
   and its output is compared against a checked-in reference file in
   `tests/golden/`. Binary MIDI output is piped through `mftext` to produce
   diffable text. Volatile lines (version banners, dates, temp paths) are
   stripped before comparison.

### 4.1 Run the tests

```sh
# All tests (Release)
ctest --preset default

# All tests (Debug)
ctest --preset debug

# Only smoke tests
ctest --test-dir build/default -L smoke

# Only golden tests
ctest --test-dir build/default -L golden
```

### 4.2 Regenerating golden files

After an **intentional** behavioural change, regenerate the references:

```sh
# Option A: via the convenience target
cmake --build build/debug --target update-golden

# Option B: via environment variable
ABCMIDI_UPDATE_GOLDEN=1 ctest --preset debug
```

Then review the diff of `tests/golden/*.txt` and commit the updated files.

---

## 5. Changes Required to Build on Windows (MSVC)

The upstream source was written primarily for GCC/Unix. Building it with
**MSVC (Visual Studio 2022)** initially failed for three reasons. The fixes
below were applied to make the CMake build succeed on Windows.

### 5.1 Linking the math library (`m.lib` does not exist on Windows)

**Problem.** Every executable target linked the Unix math library `m`:

```cmake
target_link_libraries(abc2midi PRIVATE
  obj_parseabc obj_parser2 obj_midifile abcmidi_common m)
```

On Windows there is no `m.lib`; the math functions (`sin`, `cos`, `sqrt`, …)
are part of the C runtime. The linker failed with:

```
LINK : fatal error LNK1181: cannot open input file 'm.lib'
```

**Fix.** In `CMakeLists.txt`, the math library is now selected conditionally:

```cmake
# The math library is only needed on non-Windows platforms; on Windows the
# math functions are part of the CRT, so linking "m" would fail.
if(WIN32)
  set(ABCMIDI_MATH_LIB "")
else()
  set(ABCMIDI_MATH_LIB m)
endif()
```

and every `target_link_libraries(... m)` was changed to
`target_link_libraries(... ${ABCMIDI_MATH_LIB})`. On Windows the variable
expands to nothing; on Unix it expands to `m`.

### 5.2 `snprintf` macro conflict with the UCRT `stdio.h`

**Problem.** Several source files contained:

```c
#ifdef _MSC_VER
#define snprintf _snprintf
#endif
```

placed **before** `#include <stdio.h>`. The Universal CRT `stdio.h` contains a
guard that emits a hard error if `snprintf` is already defined as a macro:

```
error C1189: #error: Macro definition of snprintf conflicts with
             Standard Library function declaration
```

This macro is obsolete: MSVC has shipped a standards-conformant C99
`snprintf` since Visual Studio 2015, so the remap to `_snprintf` is no longer
needed (and is slightly less correct, since `_snprintf` does not guarantee
NUL-termination).

**Fix.** Removed the `#define snprintf _snprintf` line from the following
files (the surrounding `#ifdef _MSC_VER` blocks and other defines such as
`ANSILIBS`, `strncasecmp`, `strcasecmp` were left intact):

- `midi2abc.c`
- `midistats.c`
- `store.c`
- `genmidi.c`
- `drawtune.c`
- `parseabc.c`
- `yapstree.c`

### 5.3 Stray semicolon in a struct definition (`midistats.c`)

**Problem.** The `eventstruc` definition in `midistats.c` had a stray
semicolon inside the brace list:

```c
struct eventstruc {int onsetTime;
                   unsigned char channel;
                   unsigned char pitch;
                   unsigned char velocity;
                   ;} midievents[50000];
```

The extra `;` is a C constraint violation. MSVC recovered from it by
**dropping the `midievents` declaration**, which then produced a large cascade
of downstream errors such as:

```
error C2065: 'midievents': undeclared identifier
error C2109: subscript requires an array or pointer type
error C2027: use of undefined type 'eventstruc'
```

**Fix.** Removed the stray semicolon:

```c
struct eventstruc {int onsetTime;
                   unsigned char channel;
                   unsigned char pitch;
                   unsigned char velocity;
                   } midievents[50000];
```

### 5.4 Summary of changed files

| File          | Change                                                        |
| ------------- | ------------------------------------------------------------- |
| `CMakeLists.txt` | Conditional `ABCMIDI_MATH_LIB` (empty on Windows, `m` otherwise); all `m` link references replaced with `${ABCMIDI_MATH_LIB}` |
| `midi2abc.c`  | Removed `#define snprintf _snprintf`                          |
| `midistats.c` | Removed `#define snprintf _snprintf`; removed stray `;` in `struct eventstruc` |
| `store.c`     | Removed `#define snprintf _snprintf`                          |
| `genmidi.c`   | Removed `#define snprintf _snprintf`                          |
| `drawtune.c`  | Removed `#define snprintf _snprintf`                          |
| `parseabc.c`  | Removed `#define snprintf _snprintf`                          |
| `yapstree.c`  | Removed `#define snprintf _snprintf`                          |

After these changes the project configures, compiles, and links cleanly on
Windows with MSVC, producing all eight executables. The remaining compiler
output consists of warnings only (deprecated CRT functions such as
`strcpy`/`fopen`, `size_t`→`int` conversions, and unused local variables) —
no errors.

---

## 6. Legacy Build (Autotools / Make, Unix)

For reference, the project also ships the original autotools build. This path
targets GCC/Unix and is **not** the path used for the Windows build.

```sh
./configure            # generates config.h and Makefile
make                   # builds all eight binaries
make install           # installs to /usr/local (default prefix)
```

Useful `configure` options:

- `--enable-debug` — compile with `-g` instead of `-O2`.

The legacy `Makefile` compiles with `-DANSILIBS -O2` and links `-lm`.

---

## 7. Troubleshooting

| Symptom | Cause | Resolution |
| ------- | ----- | ---------- |
| `LNK1181: cannot open input file 'm.lib'` | Linking `m` on Windows | Ensure the `ABCMIDI_MATH_LIB` conditional is in place (Section 5.1) |
| `C1189: Macro definition of snprintf conflicts...` | `#define snprintf _snprintf` before `#include <stdio.h>` | Remove the obsolete macro (Section 5.2) |
| `C2065: 'midievents': undeclared identifier` | Stray `;` in `struct eventstruc` | Remove the stray semicolon (Section 5.3) |
| `No configure preset is active` (CMake Tools) | IDE build tool needs an active preset | Run `cmake --preset default` in a terminal first, or select the preset in the IDE |
| Golden test fails after a code change | Output intentionally changed | Regenerate goldens (Section 4.2) and review the diff |

---

## 8. Quick Reference

```sh
# Configure + build (Release)
cmake --preset default --fresh
cmake --build build/default --config Release

# Run tests
ctest --preset default

# Regenerate golden references (after an intentional change)
cmake --build build/debug --target update-golden
```
