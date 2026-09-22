# AGENT.md

This document provides guidance for AI coding assistants working in this repository.

## Project Overview

**bee.lua** is a cross-platform Lua extension library that provides system-level native bindings for Lua 5.4 and 5.5. It wraps operating-system APIs for asynchronous I/O, networking, subprocess management, multithreading, file watching, and more, exposing them through a unified Lua interface.

## Build Commands

The project uses **luamake** (a Lua-based build system) and relies on **Ninja** underneath.

```bash
luamake                    # Build and run tests
luamake -notest            # Build only; skip tests
luamake -EXE lua           # Also build the Lua executable
luamake -arch x86_64       # Select the target architecture
luamake -mode debug        # Debug build (release is the default)
luamake -sanitize          # Enable AddressSanitizer
luamake --analyze          # Run static analysis
```

## Test Commands

```bash
luamake test               # Run tests only; do not rebuild
luamake test -v            # Run tests only, with verbose output
luamake test -v <pattern>  # Run only test cases whose names match <pattern>
```

Test files are located in `test/` and use the **ltest** framework. The `test/test.lua` entry point loads the appropriate platform-specific test files.

## Testing Requirements

The following requirements are mandatory:

- **After every code change, first build with `luamake -notest`, then run the full test suite with `luamake test -v`. Do not finish a task until both succeed.**
- **When adding or modifying an API, update the corresponding LuaLS type-annotation file under `meta/`.**
- **When adding an API, add test coverage in the corresponding file under `test/`.**
- When debugging an individual test, use `luamake test -v <pattern>` to narrow down the issue quickly. The complete test suite must still pass before the task is complete.
- If a test fails, fix the implementation until all tests pass. Do not skip or ignore failing tests.

## Code Formatting

```bash
luamake lua compile/clang-format.lua   # Format C++ code with clang-format
```

## Architecture

### Layering

```text
Lua user code
    └── Lua runtime (3rd/lua54/ or 3rd/lua55/)
        └── binding/lua_*.cpp        ← C++ ↔ Lua FFI bridge layer (one file per module)
            └── bee/*.cpp/h          ← Core C++ library (platform-independent abstractions)
                └── Operating-system APIs ← IOCP / io_uring / GCD / kqueue / epoll
```

### Key Directories

- **`bee/`** — Core C++ library. Each subdirectory corresponds to a functional domain:
  - `bee/net/` — Sockets, network endpoints, and IP utilities
  - `bee/async/` — Platform-native asynchronous I/O (Windows: IOCP; Linux: io_uring/epoll; macOS: GCD)
  - `bee/subprocess/` — Subprocess management, including stdin/stdout/stderr pipes
  - `bee/filewatch/` — Filesystem monitoring (inotify / FSEvents / ReadDirectoryChangesW)
  - `bee/thread/` — Threads, spinlocks, atomics, and semaphores
  - `bee/lua/` — Lua binding infrastructure (`binding.h`, `module.h`, and error handling)
  - `bee/sys/` — Platform abstractions for paths, file handles, and error codes
  - `bee/crash/` — Stack unwinding and crash reporting

- **`binding/`** — Contains one `lua_*.cpp` file per module. These files register Lua modules, perform type conversion, and manage userdata lifecycles.

- **`compile/`** — Lua-based build configuration. `common.lua` contains compiler flags and Lua-version selection; `make.lua` is the entry point.

- **`meta/`** — EmmyLua type annotations for IDE support, documenting the public API of each `bee.*` module.

- **`test/`** — Lua test scripts, with one test file per module.

- **`3rd/`** — Third-party dependencies: `lua54/`, `lua55/`, `fmt/` (formatting), `filesystem.h` (backport), and `lua-seri` (serialization).

- **`bootstrap/`** — A standalone launcher executable that preloads the bee library.

### Platform-Specific Code Pattern

Platform differences should be separated by file naming rather than `#ifdef` wherever practical. For example, asynchronous I/O is split into dedicated files:

- `async_uring_linux.cpp` — Linux using io_uring
- `async_epoll_linux.cpp` — Linux epoll fallback
- `async_macos.cpp` — macOS using GCD
- `async_win.cpp` — Windows using IOCP

### Lua Modules

All modules are in the `bee.*` namespace:

`socket`, `subprocess`, `thread`, `async`, `filesystem`, `filewatch`, `channel`, `epoll`, `select`, `serialization`, `time`, `crash`, `debugging`, `platform`, `sys`, `windows`

### Custom Lua Patches

The vendored Lua sources are patched; see `3rd/lua-patch/`. Patches applicable to both Lua 5.4 and Lua 5.5 include:

- ANSI escape-sequence support on Windows
- UTF-8 string encoding on Windows
- Faster `setjmp` usage on Windows
- Error/resume/yield hooks for debugger integration
- Tail-call disabling in debug mode
- `lua_assert` enabled in debug builds

## CI Matrix

The test matrix covers Windows (x86, x86_64, Clang, and MinGW), macOS (multiple versions on Intel and ARM), Linux (Ubuntu 22.04/24.04 and ARM), FreeBSD, OpenBSD, NetBSD, and ARMv7/RISC-V through QEMU. See `.github/workflows/test.yml` for the complete matrix.
