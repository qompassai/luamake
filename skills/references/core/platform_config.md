# Platform- and Compiler-Specific Configuration

Target properties support conditional overrides by platform or compiler. Nest a child table named after a platform or compiler inside the target properties. During the build, `luamake` automatically selects the table that matches the current environment and merges it into the main properties.

## Platform-Specific Configuration

Supported platform names match the possible values of `lm.os`:

| Platform key | Description |
|---|---|
| `windows` | Windows |
| `linux` | Linux |
| `macos` | macOS |
| `ios` | iOS |
| `android` | Android |
| `freebsd` | FreeBSD |
| `openbsd` | OpenBSD |
| `netbsd` | NetBSD |

```lua
lm:exe "myapp" {
    sources = "src/*.cpp",
    windows = {
        sources = "src/win/*.cpp",
        links = { "ws2_32", "user32" },
        defines = "WIN32_LEAN_AND_MEAN",
    },
    linux = {
        sources = "src/linux/*.cpp",
        links = { "pthread", "dl" },
    },
    macos = {
        sources = "src/mac/*.mm",
        frameworks = { "Foundation", "Cocoa" },
    },
}
```

## Compiler-Specific Configuration

Supported compiler names match the possible values of `lm.compiler`:

| Compiler key | Description |
|---|---|
| `msvc` | Microsoft Visual C++ |
| `gcc` | GCC |
| `clang` | Clang |
| `clang_cl` | Clang-CL (Clang in MSVC-compatible mode on Windows) |
| `mingw` | MinGW (GCC on Windows; active when `lm.os == "windows"` and the shell is `sh`) |
| `emcc` | Emscripten |

```lua
lm:exe "myapp" {
    sources = "src/*.cpp",
    msvc = {
        flags = "/utf-8",
        ldflags = "/SUBSYSTEM:CONSOLE",
    },
    gcc = {
        flags = { "-Wno-unused-parameter" },
        links = "stdc++fs",
    },
    clang = {
        flags = { "-Wno-unused-parameter" },
    },
}
```

## Merge Rules

Platform and compiler configuration can be used together. `luamake` merges properties in this order:

1. Main properties (the shared configuration).
2. The matching platform child table, such as `windows`.
3. The matching compiler child table, such as `msvc`.
4. Special case: `mingw` applies when `lm.os == "windows"` and the shell is `sh`.
5. Special case: `clang_cl` applies when `lm.compiler == "msvc"` and `cc` is `clang-cl`.

List-like properties—such as `sources`, `defines`, and `links`—are **appended** to the main property. Scalar properties **override** the main property.

## Compilation Option Properties

The following properties control compilation behavior. They can be set on a target or in `lm:conf`.

| Property | Type | Allowed values | Default | Description |
|---|---|---|---|---|
| `c` | string | `"c89"`, `"c99"`, `"c11"`, `"c17"`, `"c23"`, `"clatest"` | `""` (compiler default) | C language standard |
| `cxx` | string | `"c++11"`, `"c++14"`, `"c++17"`, `"c++20"`, `"c++23"`, `"c++latest"` | `""` (compiler default) | C++ language standard |
| `warnings` | string | `"off"`, `"on"`, `"all"`, `"error"`, `"strict"` | `"on"` | Warning level |
| `optimize` | string | `"off"`, `"size"`, `"speed"`, `"maxspeed"` | `"off"` in debug mode; otherwise `"speed"` | Optimization level |
| `mode` | string | `"debug"`, `"release"` | `"release"` | Build mode |
| `crt` | string | `"dynamic"`, `"static"` | `"dynamic"` | C runtime linkage |
| `visibility` | string | `"hidden"`, `"default"` | `"hidden"` | Symbol visibility on non-Windows platforms |
| `rtti` | string | `"on"`, `"off"` | `"on"` | C++ runtime type information |
| `lto` | string | `"on"`, `"off"`, `"thin"` (Clang only) | `"on"` for MSVC release builds; otherwise `"off"` | Link-time optimization |
| `permissive` | string | `"on"`, `"off"` | `"off"` | MSVC permissive mode; MSVC only |

## Cross-Compilation with Clang

Clang supports cross-compilation through the `target`, `arch`, `vendor`, and `sys` properties:

```lua
-- Option 1: Specify a target triple directly.
lm:conf {
    target = "aarch64-linux-gnu",
}

-- Option 2: Specify each target component separately.
lm:conf {
    arch = "aarch64",
    vendor = "linux",
    sys = "gnu",
}

-- Minimum supported macOS version.
lm:conf {
    sys = "macos14.0",   -- Equivalent to -mmacosx-version-min=14.0
}

-- Minimum supported iOS version.
lm:conf {
    sys = "ios17.0",     -- Equivalent to -miphoneos-version-min=17.0
}

-- Architecture only, for universal macOS/iOS binaries.
lm:conf {
    arch = "arm64",
}
```

## MSVC-Specific Features

### `msvc_copydll`

Copy MSVC runtime DLLs to a specified directory:

```lua
lm:msvc_copydll "copy_vcrt" {
    type = "vcrt",        -- "vcrt" | "ucrt" | "asan"
    outputs = "$bin",
}
```

| `type` | Description |
|---|---|
| `vcrt` | Visual C++ runtime DLL |
| `ucrt` | Universal C Runtime DLL |
| `asan` | AddressSanitizer DLL |
