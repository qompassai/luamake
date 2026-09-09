---
name: luamake
description: Luamake build system guide—for the current project’s `luamake` / `make.lua` / Ninja-generation workflow. Use this skill when the user needs to write, modify, troubleshoot, or understand `make.lua`, target definitions, `lm:conf`, `deps` / `objdeps`, code generation, Lua C modules, Bee runtime integration, or build problems driven by `luamake` in the current project. Use this skill even when the user does not explicitly mention `luamake` if the context clearly concerns this project’s build scripts, build directories, target dependencies, or LuaMake-generated Ninja workflow. Do not use it for purely general C/C++ compilation knowledge, unrelated CMake / xmake / Bazel / Meson problems, or compiler-theory discussion that does not involve `luamake` / `make.lua`.
---

# Luamake Build System

A Lua-based build system that generates Ninja build files.

## Quick Start

```lua
local lm = require "luamake"

lm:exe "myapp" {
    sources = { "src/*.c", "!src/test.c" },
    includes = "include",
    defines = { "DEBUG", "VERSION=1.0" },
}
```

```bash
luamake          # Build
luamake clean    # Clean
luamake rebuild  # Rebuild
```

---

## Target Types

| API | Description | Output |
|-----|-------------|--------|
| `lm:exe` | Executable | `app.exe` / `app` |
| `lm:dll` | Dynamic library | `lib.dll` / `lib.so` / `lib.dylib` |
| `lm:lib` | Static library | `lib.lib` / `lib.a` |
| `lm:source_set` | Source set; does not link independently and is reused through `deps` | None |
| `lm:lua_src` | Lua C module statically embedded into `lm:lua_exe` | None |
| `lm:lua_dll` | Dynamically loaded Lua C module | `module.dll` / `module.so` |
| `lm:lua_exe` | Executable with embedded Lua | `app.exe` / `app` |
| `lm:lua_embed` | Embeds Lua resources by named group and generates a source set; automatically exports `includes` / `objdeps` | None; produces a `source_set` |
| `lm:phony` | Phony target that aggregates dependencies | None |

---

## Target Properties

### Source Files

| Property | Description | Example |
|----------|-------------|---------|
| `sources` | Source files | `"src/*.cpp"`, `{ "a.cpp", "b.cpp" }` |
| `includes` | Include directories | `"include"`, `{ "include", "3rd" }` |
| `sysincludes` | System include directories; suppresses warnings from included headers | `"/usr/local/include"` |
| `defines` | Preprocessor definitions | `{ "DEBUG", "VER=1" }` |
| `objdeps` | Object dependencies that must be generated before compilation | `"generated_header"` |

> **Using `objdeps`:** When source files depend on generated output, such as a generated header, declare the generating target through `objdeps`. This ensures that generation occurs before compilation. `deps` controls link/build dependency order, but does not guarantee that generated files exist before source compilation.
>
> ```lua
> lm:rule "gen_header" { ... }  -- A rule that generates a header file
>
> lm:exe "app" {
>     sources = "src/*.cpp",
>     objdeps = "gen_header",  -- Generate the header before compiling src/*.cpp
> }
> ```
>
> **Note:** An `lm:lua_embed` target automatically exports its `includes` and `objdeps`. A target that depends on it through `deps` does not need to declare those properties manually. See the `lm:lua_embed` section below.

### Linking

| Property | Description | Example |
|----------|-------------|---------|
| `links` | Libraries to link | `{ "pthread", "ws2_32" }` |
| `linkdirs` | Library search directories | `"lib"`, `{ "lib", "/usr/lib" }` |
| `ldflags` | Linker flags | `"-Wl,--as-needed"` |
| `frameworks` | macOS/iOS frameworks | `{ "Foundation", "Cocoa" }` |
| `deps` | Target dependencies | `"lib1"`, `{ "lib1", "lib2" }` |

### Compilation

| Property | Description | Example |
|----------|-------------|---------|
| `flags` | Compiler flags | `{ "-Wall", "-O2" }` |
| `cflags` | C compiler flags | `"-std=c11"` |
| `cxxflags` | C++ compiler flags | `"-std=c++20"` |
| `confs` | References a named configuration | `"myconfig"` |

> **Note:** `cflags` and `cxxflags` are passed directly to the compiler command line. Within `lm:conf`, you can use `c = "c11"` and `cxx = "c++20"` as shorthand; LuaMake automatically converts them into the corresponding compiler-specific standard flags, such as MSVC’s `/std:c11`.

### Special Target Usage

**`lm:source_set`** — Organizes source files into a reusable source set. It is reused through `deps` and is not linked independently:

```lua
-- Define shared source files.
lm:source_set "common" {
    includes = "include",
    sources = { "src/common/*.cpp" },
}

-- Reuse the same sources in multiple targets.
lm:exe "app1" {
    deps = "common",
    sources = "src/app1.cpp",
}

lm:exe "app2" {
    deps = "common",
    sources = "src/app2.cpp",
}
```

**`lm:lua_src`** — Statically embeds a Lua C module into an `lm:lua_exe`:

```lua
lm:lua_src "mymodule" {
    sources = "src/mymodule.c",
}

lm:lua_exe "app" {
    deps = "mymodule",
    sources = "src/main.cpp",
}
```

**`lm:lua_embed`** — Embeds Lua files into C source by named group, producing a `source_set` that other targets can depend on. Each key under `data` is a group; the group name becomes a field of the generated `lua_embed_bundle` structure. Each group can independently set `bytecode = true`.

`bee_glue = true` enables the Bee glue layer. Under this convention, `data.preload` injects modules into `_PRELOAD`, and `data.main[1]` is used as the entry point. The target automatically exports `includes` and `objdeps`, so a dependent target does not need to declare either manually when using `deps`.

For complete rules—including group semantics, `pattern` syntax, bytecode tradeoffs, the `bee_glue` contract, and integration without the glue layer—see `references/advanced/lua_embed.md`.

```lua
lm:lua_embed "myembed" {
    bee_glue = true,
    data = {
        main = {
            bytecode = true,
            "src/main.lua",
        },
        preload = {
            bytecode = true,
            {
                dir = "lualib",
            },
        },
        data = {
            {
                name = "config.json",
                file = "assets/config.json",
            },
        },
    },
}

lm:lua_src "glue" {
    deps = "myembed", -- Automatically receives includes and objdeps.
    includes = "3rd/bee.lua",
    sources = "src/glue.cpp",
}

lm:lua_exe "app" {
    deps = { "glue", "myembed" },
    sources = "src/main.cpp",
}
```

**`lm:phony`** — Aggregates multiple targets under a named dependency for convenient reuse:

```lua
lm:phony "all_libs" {
    deps = { "libfoo", "libbar", "libqux" },
}

lm:exe "app" {
    deps = "all_libs",
    sources = "main.cpp",
}
```

---

## Configuration (`lm:conf`)

`lm:conf` has two forms with completely different behavior. Do not confuse them.

### Anonymous Configuration: `lm:conf { ... }`

Calling `lm:conf` without a name applies properties immediately to all subsequent targets in the current workspace. It acts as a global default.

```lua
-- Every option is optional. Use only the options your project needs.
lm:conf {
    c = "c11",             -- C standard: "c89", "c99", "c11", "c17"
    cxx = "c++20",         -- C++ standard: "c++11", "c++14", "c++17", "c++20"
    visibility = "hidden", -- Symbol visibility: "hidden" or "default"
}

-- These targets automatically inherit the preceding configuration.
lm:exe "app1" {
    sources = "src/app1.cpp",
}

lm:exe "app2" {
    sources = "src/app2.cpp",
}
```

**Use this when:** Most project targets share the same foundational settings, such as language standards or build-mode defaults.

### Named Configuration: `lm:conf "name" { ... }`

Calling `lm:conf` with a name does **not** apply configuration automatically. It stores the configuration for explicit use by targets through the `confs` property.

```lua
-- Defines a named configuration but applies it to no target automatically.
lm:conf "mylib" {
    defines = "MYLIB_API",
    includes = "3rd/mylib",
}

-- Only this target receives the named configuration.
lm:exe "app" {
    confs = "mylib",
    sources = "main.cpp",
}
```

**Use this when:** Several specific targets need the same include directories or preprocessor definitions, but those settings should not affect the whole project.

### Choosing a Form

| Situation | Recommended form |
|-----------|------------------|
| Set project-wide C/C++ standards, build mode, or base configuration | `lm:conf { ... }` |
| Share a specific set of include paths or definitions across selected targets | `lm:conf "name" { ... }` with `confs = "name"` |
| Only one target needs special settings | Put properties directly on that target; do not create a `conf` |

---

## Built-In Variables

### Path Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `$builddir` | Build directory; defaults to `build` and can be customized | `build` |
| `$bindir` | Binary output directory | `build/bin` |
| `$objdir` | Object output directory | `build/obj` |
| `$bin` | Alias for `$bindir` | |
| `$obj` | Alias for `$objdir` | |
| `$in` | Rule input file | |
| `$out` | Rule output file | |

> **Custom build directory:** If the project already uses a `build` directory, change LuaMake’s output directory near the beginning of the script:
>
> ```lua
> lm.builddir = "_build" -- Build outputs are written to _build/
> ```
>
> `$bindir` and `$objdir` are based on `$builddir` by default and update automatically when it changes.

### Global Variables

| Variable | Description | Example values |
|----------|-------------|----------------|
| `lm.os` | Target operating system | `"windows"`, `"linux"`, `"macos"`, `"ios"`, `"android"` |
| `lm.compiler` | Compiler | `"msvc"`, `"gcc"`, `"clang"` |
| `lm.arch` | Architecture | `"x86_64"`, `"x86"`, `"arm64"` |
| `lm.mode` | Build mode | `"debug"`, `"release"` |
| `lm.workdir` | Working directory | Project root |

---

## Development Workflow

After writing or modifying a `make.lua` build script, you **must** run a build command to verify that the script is correct:

```bash
# 1. Build to detect syntax errors and configuration problems.
luamake

# 2. If the build fails, fix the error shown and build again.
luamake

# 3. Rebuild from a clean state to rule out incremental-build interference.
luamake rebuild
```

### Notes

- **Windows:** Run `luamake` from **PowerShell**, not Git Bash.
- Run `luamake` after every `make.lua` change. Do not write the script without verifying it.
- Review both build warnings and errors; they usually identify configuration problems such as invalid paths or missing dependencies.
- When a build fails, analyze its output, fix the reported issue, and run the build again until it succeeds.

---

# Best Practices

| Topic | Document | Contents |
|-------|----------|----------|
| Dependency management | `references/best_practices/bp_dependency.md` | `deps` declaration order, source-set reuse, incremental source sets, dynamic module discovery |
| Conditional compilation and static analysis | `references/best_practices/bp_compilation.md` | Conditional compilation and static-analysis integration |
| Code generation and `objdeps` | `references/best_practices/bp_codegen.md` | Code-generation pipelines and use of `objdeps` |
| Lua modules and tests | `references/best_practices/bp_lua_and_test.md` | Lua C modules and the built-in test framework |

### Additional Detailed Documentation

| Topic | Document |
|-------|----------|
| Bee runtime library, including API-reference sources, module inventory, and Meta-file usage | `references/advanced/bee_runtime.md` |
| Authoritative `lm:lua_embed` reference, including groups, patterns, bytecode, `bee_glue`, and custom integration | `references/advanced/lua_embed.md` |

---

## Bee Runtime Library

When you run a script with `luamake lua script.lua`, LuaMake preloads the Bee library. See `references/advanced/bee_runtime.md` for API-reference sources and the module inventory.

### Scope of This Skill

Prefer this skill in the following situations:

- The current repository explicitly uses `luamake` or defines build targets through `make.lua`.
- The user is modifying `make.lua`, troubleshooting a `luamake` error, or learning how `deps`, `objdeps`, `lm:conf`, target types, build directories, or related mechanisms work.
- The issue directly concerns the project’s Ninja-generation workflow, Bee integration, or Lua C module build process.

This skill should generally **not** be used for:

- General C/C++ compilation, linking, or optimization theory unrelated to the current project.
- Pure CMake, xmake, Bazel, Meson, or other build-system questions.
- Compiler syntax, language-standard, or platform-API questions that do not lead back to `luamake` or `make.lua`.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| “No source files found” | Check glob patterns, confirm `rootdir`, and verify source-file extensions |
| Windows linking error | Add required system libraries, for example: `links = { "ws2_32", "iphlpapi", "user32" }` |
| Cross-compilation problem | Use platform-specific configuration and check `lm.compiler` |
| Generated file is missing | Add `objdeps` so generation completes before compilation |
| `build` directory conflict | If the project already uses a `build` directory, set `lm.builddir = "_build"` |
| Header file cannot be found | Check `includes` paths; use absolute paths or `$builddir` when appropriate |
| Ninja build failure | Delete the `build/` directory and rebuild |
