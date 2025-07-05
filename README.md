# SRAL
Screen Reader Abstraction Library
## Description
SRAL is a cross-platform library for output text using speech engines.

## Platforms
SRAL is supported on Windows, MacOS and Linux platforms.

## Header
See how to use SRAL in Include/SRAL.h

## Compilation

SRAL uses CMake for cross-platform building with flexible configuration options.

### Build Options

- `BUILD_SHARED_LIBS` - Build shared library (ON) or static library (OFF). Default: ON
- `BUILD_TESTING` - Build test executables. Default: ON  
- `BUILD_EXAMPLES` - Build example programs. Default: ON

**Windows-only options:**

- `SRAL_USE_STATIC_CRT` - Use static C runtime library (/MT). Default: OFF
- `SRAL_ENABLE_UIA` - Enable UI Automation support. Default: ON

### Quick Build

**Shared library (default):**

```bash
cmake . -B build
cmake --build build --config Release
```

**Static library only:**

```bash
cmake . -B build -DBUILD_SHARED_LIBS=OFF
cmake --build build --config Release
```

**Minimal build (static library, no tests/examples):**

```bash
cmake . -B build -DBUILD_SHARED_LIBS=OFF -DBUILD_TESTING=OFF -DBUILD_EXAMPLES=OFF
cmake --build build --config Release
```

**Windows-specific builds:**

```bash
# Static library with static CRT (/MT) - for static linking
cmake . -B build -DBUILD_SHARED_LIBS=OFF -DSRAL_USE_STATIC_CRT=ON
cmake --build build --config Release

# Static library with dynamic CRT (/MD) - for linking with /MD projects  
cmake . -B build -DBUILD_SHARED_LIBS=OFF -DSRAL_USE_STATIC_CRT=OFF
cmake --build build --config Release

# Disable UIA support to avoid linking issues
cmake . -B build -DBUILD_SHARED_LIBS=OFF -DSRAL_ENABLE_UIA=OFF
cmake --build build --config Release
```

### Build Outputs

- **Shared build**: `libSRAL.so` (Linux), `SRAL.dll` (Windows), `libSRAL.dylib` (macOS)
- **Static build**: `libSRAL.a` (Linux), `SRAL.lib` (Windows)
- **Test executable**: `SRAL_test` (if BUILD_TESTING=ON)
- **Headers**: `Include/SRAL.h`

### Platform Requirements

**Linux:**

```bash
sudo apt install libspeechd-dev libbrlapi-dev brltty pkg-config
```

**Windows:**

- Visual Studio 2019 or newer
- Windows SDK

**macOS:**

- Xcode command line tools


## Support for NVDAControlEx

SRAL supports the [NVDAControlEx](https://github.com/m1maker/NVDAControlEx) add-on, allowing developers to extended manage the NVDA functions.

## Usage

To use the SRAL API in a C/C++ project, you need a statically linked or dynamically imported SRAL library, as well as a SRAL.h file with function declarations.
If you use SRAL as a static library for Windows, you need to define SRAL_STATIC in the SRAL.h before the include

```c
#define SRAL_STATIC
#include <SRAL.h>
```

## Additional info

For [NVDA](https://github.com/nvaccess/nvda) screen reader, you need to download the [Controller Client](https://www.nvaccess.org/files/nvda/releases/stable/). We don't support old client V 1.
