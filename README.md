# taupkg

The official package manager for the [Tauraro](https://github.com/tauraro/tauraro) programming language.

taupkg is a single native binary with no runtime dependencies. It handles
dependency resolution, downloading, building, and publishing Tauraro packages.

---

## Table of contents

- [Installation](#installation)
- [Quick start](#quick-start)
- [taupkg.toml reference](#taupkgtoml-reference)
- [taupkg.lock](#taupkglock)
- [Commands](#commands)
  - [init](#init)
  - [add](#add)
  - [remove](#remove)
  - [install](#install)
  - [update](#update)
  - [build](#build)
  - [run](#run)
  - [test](#test)
  - [list](#list)
  - [info](#info)
  - [search](#search)
  - [publish](#publish)
  - [clean](#clean)
  - [install-tauraro](#install-tauraro)
- [Dependency sources](#dependency-sources)
- [Version constraints](#version-constraints)
- [Project layout](#project-layout)
- [Building from source](#building-from-source)

---

## Installation

### Download a pre-built binary

Download the latest release for your platform from the
[releases page](https://github.com/tauraro/taupkg/releases):

| Platform | Binary |
|----------|--------|
| Linux x64 | `taupkg-linux-x64.zip` |
| Linux ARM64 | `taupkg-linux-arm64.zip` |
| Windows x64 | `taupkg-windows-x64.zip` |
| macOS ARM64 | `taupkg-macos-arm64.zip` |

Unzip and place `taupkg` (or `taupkg.exe`) somewhere on your `PATH`.

### Install via taupkg itself

If you already have taupkg, use it to install or update the Tauraro compiler:

```sh
taupkg install-tauraro
```

### Build from source

See [Building from source](#building-from-source).

---

## Quick start

```sh
# Create a new project
taupkg init myproject
cd myproject

# Add a dependency
taupkg add mathlib@^1.0.0

# Build
taupkg build

# Build and run
taupkg run
```

---

## taupkg.toml reference

Every taupkg project has a `taupkg.toml` manifest at its root.

```toml
[package]
name    = "myproject"
version = "0.1.0"
desc    = "A short description of the project"
license = "MIT"
bin     = "src/main.tr"       # entry-point source file
authors = ["Alice <alice@example.com>"]

[deps]
# Version constraints (see Version constraints section)
mathlib   = "^1.2.0"
httplib   = "~0.9.1"
utilities = "*"

# Pin to an exact version
logger = "2.0.0"

[build-deps]
# Dependencies only needed at compile time (not shipped to end users)
codegen = "^0.3.0"

[sources]
# Override the source for a specific package.
# If omitted, packages are fetched from the official registry.
mathlib   = "github:user/mathlib@v1.2.3"
utilities = "local:../my-utilities"
httplib   = "git:https://gitlab.com/org/httplib.git@main"
```

### `[package]` fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | **yes** | Package name (lowercase, hyphens allowed) |
| `version` | **yes** | Semantic version: `MAJOR.MINOR.PATCH` |
| `desc` | no | Short description shown in registry search |
| `license` | no | SPDX licence identifier (default: `MIT`) |
| `bin` | no | Entry-point `.tr` file (default: `src/main.tr`) |
| `authors` | no | Author name(s) |

### `[deps]` / `[build-deps]`

Map of package name → version constraint. See [Version constraints](#version-constraints).

### `[sources]`

Override the fetch location for a package. Supported forms:

| Form | Example |
|------|---------|
| GitHub shorthand | `github:user/repo@v1.2.3` |
| Arbitrary git URL | `git:https://host/repo.git@branch` |
| Local path | `local:../sibling-package` |

---

## taupkg.lock

`taupkg.lock` is written automatically. It pins every resolved dependency to an
exact version, source URL, and SHA-256 checksum. **Commit this file** to get
reproducible builds across machines and CI.

```toml
# taupkg lock file -- do not edit manually

[pkg]
name     = "mathlib"
version  = "1.2.3"
source   = "github:user/mathlib@v1.2.3"
checksum = "sha256:a3f..."
```

To force a fresh resolution (discard all pins), delete `taupkg.lock` and run
`taupkg install`.

---

## Commands

### init

```sh
taupkg init [name]
```

Create `taupkg.toml` and `src/main.tr` in the current directory. If `name` is
omitted the directory name is used (lowercased).

```
Created taupkg.toml and src/main.tr
```

---

### add

```sh
taupkg add <package>[@constraint]
taupkg add <package> <source-uri>
```

Download and install a package, then write it to `[deps]` in `taupkg.toml` and
update `taupkg.lock`.

```sh
taupkg add mathlib              # latest version
taupkg add mathlib@^1.2.0       # compatible with 1.x
taupkg add mathlib@~1.2.0       # patch updates only
taupkg add mathlib@2.0.0        # exact version
taupkg add mathlib github:alice/mathlib@v2.0.0   # custom source
```

---

### remove

```sh
taupkg remove <package>
```

Remove a package from `taupkg.toml`, delete its files from `.taupkg/packages/`,
and update `taupkg.lock`.

---

### install

```sh
taupkg install [--verbose]
```

Install all packages listed in `taupkg.toml`. Uses locked versions from
`taupkg.lock` when they satisfy the constraint. Writes an updated lock file.

```
Installing mathlib 1.2.3...
  + mathlib@1.2.3
All packages installed.
```

---

### update

```sh
taupkg update [package] [--verbose]
```

Re-fetch and update all deps (or a single named dep) to the newest version
that satisfies the constraint, then update `taupkg.lock`.

---

### build

```sh
taupkg build [--release] [-o output]
```

Compile the project entry point (from `taupkg.toml` `bin` field) into a binary.

| Flag | Description |
|------|-------------|
| `--release` | Compile with `-O3` optimisations |
| `-o <file>` | Override the output binary path |

The binary and intermediate C files are written to `.taupkg/build/`.

---

### run

```sh
taupkg run [--release] [-- <args>]
```

Build then immediately run the binary. Arguments after `--` are passed through
to the program.

```sh
taupkg run -- --port 8080
```

---

### test

```sh
taupkg test [--verbose]
```

Compile and run test files found in the `tests/` directory. Test files must
be named `test_*.tr`.

---

### list

```sh
taupkg list
```

Print all packages currently recorded in `taupkg.lock`.

```
Installed packages:
  mathlib 1.2.3
  httplib 0.9.4
```

---

### info

```sh
taupkg info <package>
```

Show detailed information about an installed package.

```
Name:     mathlib
Version:  1.2.3
Source:   github:alice/mathlib@v1.2.3
Checksum: sha256:a3f9c2...
```

---

### search

```sh
taupkg search <query>
```

Search the official Tauraro package registry.

---

### publish

```sh
taupkg publish [--verbose]
```

Create a source tarball of your project's `src/` directory and publish it to
the official registry. Requires a registry token.

> **Note**: `publish` creates a `.tar.gz` of `src/` — not the compiled binary.
> Publishing is not supported on Windows yet; use WSL or `git archive` instead.

---

### clean

```sh
taupkg clean
```

Delete the `.taupkg/build/` directory (compiled output). Does not touch
downloaded packages in `.taupkg/packages/`.

---

### install-tauraro

```sh
taupkg install-tauraro [--version x.y.z] [--mirror] [--verbose]
```

Download and install the Tauraro compiler from GitHub releases (or
`tauraro.org` mirror) for the current platform.

| Flag | Description |
|------|-------------|
| `--version x.y.z` | Install a specific version (default: latest) |
| `--mirror` | Use `tauraro.org` mirror instead of GitHub |
| `--verbose` | Show download progress and paths |

The compiler is installed to `~/.taupkg/bin/tauraroc-{os}-{arch}/` and the
directory is added to your `PATH` automatically.

**Supported platforms**: linux-x64, linux-arm64, windows-x64, macos-arm64.

```sh
taupkg install-tauraro
# → Installing tauraroc 0.0.4 (linux-x64)...
# →   OK: tauraroc v0.0.4
# → Installed:
# →   binary:  ~/.taupkg/bin/tauraroc-linux-x64/tauraroc
# →   runtime: ~/.taupkg/bin/tauraroc-linux-x64/runtime/tauraro_rt.h
# →   std:     ~/.taupkg/bin/tauraroc-linux-x64/std/
```

---

## Dependency sources

When no `[sources]` override is given, packages are fetched from the official
registry. To use a different source, add an entry to `[sources]`:

### GitHub

```toml
[sources]
mypkg = "github:alice/mypkg@v1.2.3"   # specific tag
mypkg = "github:alice/mypkg@main"      # branch
mypkg = "github:alice/mypkg"           # default branch
```

Internally resolved to `https://github.com/alice/mypkg.git` and cloned with
`git clone --depth 1`.

### Arbitrary git

```toml
[sources]
mypkg = "git:https://gitlab.com/org/mypkg.git@v2.0.0"
```

### Local path

Useful during development of related packages:

```toml
[sources]
mypkg = "local:../mypkg"          # relative path
mypkg = "local:/home/dev/mypkg"   # absolute path
```

Local packages are copied (not symlinked) into `.taupkg/packages/`.

---

## Version constraints

Version strings follow [Semantic Versioning](https://semver.org).

| Constraint | Meaning | Example range |
|------------|---------|---------------|
| `*` | Any version | — |
| `1.2.3` | Exact match | `== 1.2.3` |
| `^1.2.3` | Compatible (same major) | `>= 1.2.3 < 2.0.0` |
| `~1.2.3` | Patch-compatible (same major.minor) | `>= 1.2.3 < 1.3.0` |
| `>=1.2.3` | At least | `>= 1.2.3` |
| `>1.2.3` | Strictly greater | `> 1.2.3` |
| `<=1.2.3` | At most | `<= 1.2.3` |
| `<1.2.3` | Strictly less | `< 1.2.3` |

---

## Project layout

```
myproject/
├── taupkg.toml          # manifest
├── taupkg.lock          # lock file (commit this)
├── src/
│   └── main.tr          # entry point
├── tests/
│   └── test_core.tr     # test files (test_*.tr)
└── .taupkg/
    ├── build/           # compiled output (gitignore this)
    └── packages/        # downloaded deps (gitignore this)
```

Add `.taupkg/build/` and `.taupkg/packages/` to `.gitignore`.
Commit `taupkg.lock`.

---

## Building from source

Prerequisites: a working `tauraroc` binary and `gcc`.

```sh
git clone https://github.com/tauraro/taupkg
cd taupkg

# Emit C from the Tauraro source
tauraroc src/main.tr --emit c

# Copy the runtime header (tauraroc cannot always find it automatically)
cp "$(dirname $(which tauraroc))/runtime/tauraro_rt.h" src/build/ 2>/dev/null || \
cp ~/.taupkg/bin/tauraroc-*/runtime/tauraro_rt.h src/build/

# Linux / macOS
cd src && gcc -O2 -std=c11 -Wno-misleading-indentation -Ibuild \
  build/main.c build/module_*.c \
  build/include/std/core/*.c \
  build/include/std/io/*.c build/include/std/io.c \
  build/include/std/net/*.c build/include/std/net.c \
  build/include/std/string/*.c \
  -o build/taupkg -lm
mv src/build/taupkg ./taupkg

# Windows (Git Bash or PowerShell with gcc on PATH)
cd src && gcc -O2 -std=c11 -Wno-misleading-indentation -Ibuild \
  build/main.c build/module_*.c \
  build/include/std/core/*.c \
  build/include/std/io/*.c build/include/std/io.c \
  build/include/std/net/*.c build/include/std/net.c \
  build/include/std/string/*.c \
  -o build/taupkg.exe -lm -lws2_32
```

---

## License

MIT — see [LICENSE](LICENSE) for details.
