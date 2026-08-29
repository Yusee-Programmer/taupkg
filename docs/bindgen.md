# FFI bindings with `taupkg bindgen`

`taupkg bindgen` generates Tauraro FFI bindings for a C or C++ header and wraps
them in a **ready-to-import taupkg package** — manifest, source, guidance, and
provenance included. It is a thin, intelligent layer over `tauraroc bindgen`.

```sh
taupkg bindgen <header> [flags...] [--name <pkg>] [--out <dir>]
```

---

## Why a package (not just a `.tr` file)?

Because a package drops straight into taupkg's [automatic module
resolution](dependency-resolution.md). You depend on it by path and import it —
no compiler flags, no manual `TAURARO_PATH`:

```toml
# your app's taupkg.toml
[deps]
mylib = { path = "./mylib" }
```

```tauraro
from mylib import some_function
```

`taupkg build` sets `TAURARO_PATH` to the generated package's `src/` and links
any C/C++ shim automatically.

---

## Generated layout

```
mylib/
├── taupkg.toml          # [package] metadata + [bindgen] provenance
├── src/
│   ├── mylib.tr         # the bindings — module name == package name
│   ├── mylib_shim.cpp   # C++→C shim (only with `-h cpp`; auto-compiled+linked)
│   └── mylib_macros.c   # macro shim (only with `--macros`)
├── guide.txt            # tailored import/build/link instructions
└── README.md
```

The bindings live at `src/<name>.tr` so `from <name> import X` resolves the
module by name once the package's `src/` is on `TAURARO_PATH` (which taupkg does
for you).

### `taupkg.toml [bindgen]` provenance

```toml
[bindgen]
header    = "mylib.h"
mode      = "c"                      # or "cpp"
linkflags = "-lz"                    # from a link pragma, if any
includes  = "stdint.h, stdio.h"      # system #includes detected
command   = "taupkg bindgen mylib.h --name mylib"   # reproducible regen
```

---

## Flags

taupkg-level flags are consumed by taupkg; everything else is forwarded verbatim
to `tauraroc bindgen`.

| Flag | Level | Description |
|------|-------|-------------|
| `--name <pkg>` | taupkg | Package name (default: header filename, sans extension) |
| `--out <dir>` | taupkg | Directory to create the package in (default: cwd) |
| `-h cpp` | forwarded | Bind a C++ header (auto C++→C shim via libclang) |
| `--macros` | forwarded | Also bind function-like C macros (compile-verified) |
| `--pkg <name>` | forwarded | Auto-discover `-I`/`-D`/`-l` via `pkg-config` |
| `-I<dir>` | forwarded | Add an include directory |
| `-D<macro>` | forwarded | Define a preprocessor macro |
| `-std=<std>` | forwarded | Language standard (e.g. `-std=c11`, `-std=c++17`) |
| `-isystem <dir>` | forwarded | Add a system include directory |
| `--cc <compiler>` | forwarded | Compiler used for parsing/shims (e.g. `clang++`) |

> `-o` is managed by taupkg (it controls the package layout) and is ignored if
> passed.

---

## The intelligent layer

After generation, taupkg analyses the result and writes a tailored `guide.txt`:

1. **`#include` scan** — reads the header and lists its **system** (`<…>`) and
   **local** (`"…"`) includes, so you can see what the library depends on. For
   well-known system headers it suggests the likely link flag
   (`zlib.h → -lz`, `sqlite3.h → -lsqlite3`, `curl/curl.h → -lcurl`, …).
2. **Pragma-aware link guidance** — parses the generated
   `# tauraro-cpp-{shim,lib,linkflags,cflags}` pragmas and states exactly what
   links automatically vs. what you must pass by hand.
3. **Self-validation** — runs `tauraroc --check` on the bindings and reports
   whether they parse and type-check (so you catch missing `-I`/`-D` early).
4. **Symbol inventory + sample import** — the top-level `def`/`class` signatures
   and a ready-to-paste `from <name> import <symbol>` line.
5. **Reproducible regeneration** — the exact `taupkg bindgen …` command, printed
   in the guide and recorded in `taupkg.toml`.

---

## Examples

### A system C library (pkg-config)

```sh
taupkg bindgen zlib.h --pkg zlib
```

`--pkg` feeds pkg-config's `--cflags` into parsing and records its `--libs` as a
link pragma, so consumers link `-lz` automatically — no manual flags.

### A C++ library

```sh
taupkg bindgen mylib.hpp -h cpp --cc clang++
```

Generates `src/mylib.tr` plus `src/mylib_shim.cpp` (an `extern "C"` bridge). The
shim is auto-compiled and linked; LTO inlines the wrapper away (zero-cost).

### A local C header + include dir

```sh
taupkg bindgen vendor/api.h -Ivendor --name api
```

---

## Linking the underlying library

| Case | What you do | What taupkg/tauraroc does |
|------|-------------|---------------------------|
| System lib via `--pkg` | nothing | records `# tauraro-cpp-linkflags`; auto-links |
| C++ header (`-h cpp`) | nothing | auto-compiles + links the shim (zero-cost via LTO) |
| Hand-vendored C source | drop `impl.c` next to the bindings and add `# tauraro-cpp-shim: impl.c` | compiles it with the **C** driver (unmangled C ABI) and links it |
| Prebuilt static/shared lib | link it: `tauraroc app.tr -l<name>` (or add to your build) | — |

For the hand-vendored C case, the compiler distinguishes a plain C source (a
`.c` file **without** `extern "C"`) from a generated shim (a `.c` with
`extern "C"`, or a `.cpp`): the former is compiled with the C compiler so its
functions keep their unmangled names, the latter with the C++ compiler.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| "the compiler ran but produced no bindings" | Your `tauraroc` is too old to support `bindgen`. Update it (`taupkg install-tauraro`) or set `TAURARO_COMPILER`. |
| "header not found" | Pass its directory with `-I<dir>`, or an absolute path. |
| C++ mode fails | Needs libclang + a C++ compiler; try `--cc clang++`. |
| Missing symbols during parse | Add the `-D<macro>` the library expects, or `--pkg`. |
| `guide.txt` says `--check` failed | The bindings need extra `-I`/`-D`; re-run with them. |

Run with `--verbose` to see the exact `tauraroc bindgen` command.
