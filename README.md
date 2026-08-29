# taupkg

The official package manager for the [Tauraro](https://github.com/tauraro/tauraro) programming language.

taupkg is a single native binary with no runtime dependencies. It handles
dependency resolution, downloading, integrity verification, building, publishing,
and — new in 0.0.2 — **automatic module-path resolution** and **FFI binding
generation** from C/C++ headers.

**Version: 0.0.2**

---

## What's new in 0.0.2

- **Smart dependency resolution** — the full transitive graph is resolved with
  **constraint accumulation across the whole graph** and **backtracking version
  selection**, unified to a single version per package. When no version can
  satisfy everyone, you get a **world-class conflict report** (every constraint,
  who imposed it, available versions, concrete fixes) — never a silent duplicate.
- **Automatic `TAURARO_PATH`** — `taupkg build`/`run`/`test` set the compiler's
  module search path automatically and compile your project **in place** (no
  sandbox copy). `from <dep> import X` just works. `taupkg env` prints the path
  for compiling with `tauraroc` directly.
- **Inline-table dependencies** — Cargo-style `{ path = … }`, `{ git = … }`,
  `{ github = … }`, `{ version = … }`, `{ …, optional = true }`, alongside the
  classic string form. Per-manifest **path rebasing** (correct for monorepos).
- **Features** — `[features]` with `--features`, `--all-features`, and
  `--no-default-features` gate optional dependencies.
- **Workspaces** — a `[workspace]` root fans `install`/`build`/`test`/`tree` out
  across member packages (with inter-member path deps).
- **Security** — content-hashed lock + `taupkg audit` (integrity **and**
  advisory/vulnerability check) + `taupkg sbom` (CycloneDX SBOM) + **ed25519
  package signing** (`taupkg key`/`sign`/`verify`, and signed `publish`).
- **Real test runner** — `taupkg test` compiles and runs every `tests/*.tr`.
- **FFI bindgen** — `taupkg bindgen` turns a C/C++ header into an importable package.
- **New commands**: `tree`, `audit`, `sbom`, `login`, `env`, `key`/`sign`/`verify`,
  and **`bindgen`**.
- **New flags**: `--locked`, `--offline`, `--sandbox`, `--target`, `--features`,
  `--all-features`, `--no-default-features`, `publish --dry-run`.

See [Dependency resolution](#dependency-resolution),
[Features](#features), [Workspaces](#workspaces), [Security](#security), and
[Generating FFI bindings](#generating-ffi-bindings-bindgen) for the details.

### In-depth guides

- [`docs/dependency-resolution.md`](docs/dependency-resolution.md) — the
  resolution model, lock file, integrity, and automatic `TAURARO_PATH`.
- [`docs/bindgen.md`](docs/bindgen.md) — generating FFI bindings as packages.

---

## Table of contents

- [Installation](#installation)
- [Quick start](#quick-start)
- [taupkg.toml reference](#taupkgtoml-reference)
- [taupkg.lock](#taupkglock)
- [Dependency resolution](#dependency-resolution)
- [Features](#features)
- [Workspaces](#workspaces)
- [Security](#security)
- [Commands](#commands)
  - [init](#init) · [add](#add) · [remove](#remove) · [install](#install)
  - [update](#update) · [build](#build) · [run](#run) · [test](#test)
  - [tree](#tree) · [audit](#audit) · [sbom](#sbom) · [env](#env) · [bindgen](#bindgen)
  - [list](#list) · [info](#info) · [search](#search) · [login](#login)
  - [key / sign / verify](#key--sign--verify) · [publish](#publish)
  - [clean](#clean) · [install-tauraro](#install-tauraro)
- [Dependency sources](#dependency-sources)
- [Version constraints](#version-constraints)
- [Generating FFI bindings (bindgen)](#generating-ffi-bindings-bindgen)
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

### Install the Tauraro compiler

taupkg drives `tauraroc`. Install or update it with:

```sh
taupkg install-tauraro
```

taupkg finds the compiler via `TAURARO_COMPILER` (explicit override) or
`tauraroc`/`tauraroc.exe` on `PATH`.

---

## Quick start

```sh
# Create a new project
taupkg init myproject
cd myproject

# Add a dependency
taupkg add mathlib@^1.0.0

# See the resolved graph
taupkg tree

# Build (resolves deps, sets TAURARO_PATH, compiles in place)
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
# Classic string form — a version constraint (registry lookup):
mathlib = "^1.2.0"
logger  = "2.0.0"             # exact version

# Inline-table form (Cargo-style):
utils   = { path = "../utils" }                       # local path
httplib = { git = "https://gitlab.com/org/httplib.git", tag = "v2.0.0" }
gfx     = { github = "alice/gfx", branch = "main" }
json    = { version = "^1.0" }                         # explicit registry

[dev-deps]
# Test/example-only dependencies — not shipped to consumers.
testkit = "^0.4.0"

[build-deps]
# Build-script-only dependencies.
codegen = "^0.3.0"

[sources]
# Optional: override the source for a package declared with a bare version.
mathlib = "github:user/mathlib@v1.2.3"

[features]            # (reserved) named feature sets
default = "std"

[workspace]           # (reserved) multi-crate workspace
members = ["crates/a", "crates/b"]
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

### Dependency sections

`[deps]` (aliases: `[dependencies]`), `[dev-deps]` (`[dev-dependencies]`), and
`[build-deps]` (`[build-dependencies]`) each map a package name to either a
version string or an inline table. See [Dependency sources](#dependency-sources).

### `[sources]`

Overrides the fetch location for a package declared with a bare version string.
(Inline-table deps carry their own source, so they don't need `[sources]`.)

---

## taupkg.lock

`taupkg.lock` is written automatically. It pins every resolved dependency —
across the **full transitive graph** — to an exact version, source, constraint,
and SHA-256 **content checksum**. **Commit this file** for reproducible builds.

```toml
# taupkg lock file -- do not edit manually

[[pkg]]
name       = "mathlib"
version    = "1.2.3"
constraint = "^1.2.0"
source     = "github:user/mathlib@<commit-sha>"
checksum   = "sha256:a3f9c2..."

[[pkg]]
name       = "libcore"
version    = "0.9.4"
constraint = "*"
source     = "local:../libcore"
checksum   = "sha256:77d8..."
```

- **Git sources are pinned to the exact commit SHA** (not just the tag), so the
  lock replays byte-for-byte even if a tag later moves.
- **Local path sources are stored project-root-relative**, so the lock survives
  a monorepo re-checkout at a different absolute location.
- `taupkg install --locked` replays the lock exactly and fails if it's stale.
- `taupkg audit` re-hashes installed packages and verifies them against the lock.

To force a fresh resolution, delete `taupkg.lock` and run `taupkg install`.

---

## Dependency resolution

taupkg builds the **entire transitive dependency graph**, then unifies every
package to a **single version**. This is deliberate: Tauraro's unity build links
exactly one copy of each module, so — unlike Cargo, which can keep multiple
semver-major versions side by side — taupkg resolves each package to one version.

To make that constraint bite as rarely as possible, the resolver **accumulates
every constraint** on a package across the whole graph, **selects the highest
version satisfying the intersection**, and re-selects/re-walks to a fixpoint when
a later constraint tightens (**backtracking version selection**). Two compatible
constraints (`^1.0` and `^1.2`) unify automatically to the newest matching
version. When no single version can satisfy everyone, you get a **world-class
conflict report** listing every constraint, who imposed it, the available/resolved
versions, and concrete fixes:

```
error: cannot select a single version of 'shared'

  'shared' is required by:
    - (root)     requires  ^2.0
    - libx 1.0.0 requires  ^1.2

  'shared' resolves to 1.5.0 from local:../shared, which does not satisfy all of
  the above ... To fix:
    - relax a constraint so a shared version exists, or
    - pin it explicitly in [deps]:  shared = "<version>", or
    - if two incompatible majors are genuinely needed, they cannot coexist in one
      build — split the dependency or vendor one under a new name.
```

### Automatic module path (`TAURARO_PATH`)

`taupkg build`, `run`, and `test` set `TAURARO_PATH` automatically to every
installed package's importable directory and then compile your project's real
`src/main.tr` **in place** — no files are copied into a sandbox. So this just
works with no compiler flags:

```tauraro
from mathlib import add      # mathlib is a [deps] entry
```

- Any pre-existing `TAURARO_PATH` in your environment is **preserved** (appended),
  so you can point at extra trees and taupkg won't clobber it.
- To compile with `tauraroc` directly (no taupkg), run [`taupkg env`](#env) and
  export the line it prints.
- If in-place resolution ever misbehaves, [`taupkg build --sandbox`](#build)
  copies everything into `.taupkg/build/` and compiles there (hermetic fallback).

### Path rebasing

A `path` dependency resolves **relative to the manifest that declares it**, not
the project root. So a dependency at `libs/liba` that declares
`libb = { path = "../libb" }` correctly finds `libs/libb`.

---

## Features

Optional dependencies are pulled in only when a feature enables them. Declare a
dependency `optional`, then wire it up under `[features]`:

```toml
[deps]
serde   = { version = "^1.0", optional = true }
openssl = { version = "^0.10", optional = true }

[features]
default = "json"           # enabled unless --no-default-features
json    = "serde"          # a feature enables an optional dep …
full    = "json, tls"      # … or other features
tls     = "openssl"
```

Select features at build/install time:

```sh
taupkg build                              # default features
taupkg build --features tls               # default + tls
taupkg build --no-default-features        # nothing optional
taupkg build --no-default-features --features full   # exactly `full`
taupkg build --all-features               # everything
```

A non-optional dependency is always built; an optional one is built only when an
enabled feature names it (transitively).

---

## Workspaces

A `[workspace]` root groups several member packages. `install`, `build`, `test`,
and `tree` fan out across every member:

```toml
# workspace-root/taupkg.toml  (a "virtual" manifest — no [package] needed)
[workspace]
members = ["crates/app", "crates/lib"]
```

```sh
taupkg build      # builds crates/app and crates/lib
```

Each member is a normal taupkg package with its own `taupkg.toml` and `src/`.
Members may depend on each other via `path` deps (`lib = { path = "../lib" }`),
which resolve correctly thanks to per-manifest path rebasing.

---

## Security

taupkg records a full-tree SHA-256 content hash of every dependency in the lock,
and offers two commands on top:

- **`taupkg audit`** verifies installed packages against those hashes (integrity)
  **and** checks them against an advisory database (vulnerabilities):

  ```
  Auditing 2 locked package(s)...
  OK: integrity verified -- all installed packages match the lock.

  Checking 2 package(s) against 1 advisory(ies)...
    ! [high] shared@1.5.0 — TAUP-2025-0001
        integer overflow in sval() before 1.6.0
        affected: <1.6.0
  ```

  Advisories are read from `.taupkg/advisories.toml` (project) or
  `~/.taupkg/advisories.toml` (global):

  ```toml
  [[advisory]]
  id          = "TAUP-2025-0001"
  package     = "shared"
  versions    = "<1.6.0"       # a version is affected if it satisfies this
  severity    = "high"
  description = "integer overflow in sval() before 1.6.0"
  ```

- **`taupkg sbom`** emits a CycloneDX 1.5 software bill of materials of the
  resolved graph (name, version, `pkg:tauraro/...` purl, SHA-256 hash, source):

  ```sh
  taupkg sbom -o sbom.json
  ```

- **Ed25519 signing** binds a publisher identity to exact content. Generate a
  keypair, and `taupkg publish` signs the package's content checksum; the
  signature + public key land in `<pkg>-<ver>.sig`:

  ```sh
  taupkg key gen                    # → ~/.taupkg/keys/default.key + a public key
  taupkg key show                   # print your public key
  taupkg publish                    # signs automatically if a key exists

  # low-level, for scripting / CI:
  taupkg sign "some message"        # → 128-hex signature
  taupkg verify "some message" <sig> <pubkey>   # VALID / INVALID (exit 0/1)

  # trust a publisher's key so their packages can be verified:
  taupkg key trust <pubkey> --name alice
  ```

  `publish` drops a `taupkg.sig` (pubkey + signature + checksum) at the package
  root; it ships with the package and is excluded from the content hash it
  certifies. On the consumer side, **`taupkg audit` verifies each signed package**
  against your trusted keys — a valid signature from a trusted key passes, a valid
  signature from an untrusted key warns, and a tampered package fails (both the
  integrity checksum and the signature catch it).

  Signatures are ed25519 (RFC 8032) via a pure-C TweetNaCl implementation in the
  Tauraro runtime; keys are 32-byte seeds (64-hex), signatures 64 bytes (128-hex).

---

## Commands

### init

```sh
taupkg init [name]
```

Create `taupkg.toml` and `src/main.tr` in the current directory. If `name` is
omitted the directory name is used (lowercased).

---

### add

```sh
taupkg add <package>[@constraint]
taupkg add <package> <source-uri>
```

Download and install a package, write it to `[deps]` in `taupkg.toml`, and
update `taupkg.lock`.

```sh
taupkg add mathlib              # latest version
taupkg add mathlib@^1.2.0       # compatible with 1.x
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
taupkg install [--locked] [--offline] [--verbose]
```

Resolve and install the **full transitive graph** from `taupkg.toml`, reusing
locked versions when they satisfy the constraints, then write an updated lock.

| Flag | Description |
|------|-------------|
| `--locked` | Replay `taupkg.lock` exactly; error if it's missing/stale (CI-friendly) |
| `--offline` | Never touch the network — reuse `.taupkg/packages/` only |

```
  + mathlib@1.2.3
  + libcore@0.9.4
Resolved 2 package(s) (direct + transitive).
```

---

### update

```sh
taupkg update [package] [--verbose]
```

Re-resolve all deps (or a single named dep) to the newest version satisfying the
constraints, then update `taupkg.lock`.

---

### build

```sh
taupkg build [--release] [--sandbox] [--target <triple>] [--features a,b] [-o output]
```

Resolve the graph, set `TAURARO_PATH`, and compile the entry point (`bin` field)
into a binary. In a `[workspace]` root, builds every member.

| Flag | Description |
|------|-------------|
| `--release` | Compile with `-O3` optimisations |
| `--sandbox` | Hermetic build: copy sources into `.taupkg/build/` and compile there |
| `--target <triple>` | Cross-compile (zero-config — tauraroc bundles `zig cc`) |
| `--features a,b` | Enable features (see [Features](#features)) |
| `--all-features` / `--no-default-features` | Enable all / none of the default features |
| `-o <file>` | Override the output binary path |

Cross-compilation examples: `--target linux-arm64`, `--target windows-x64`,
`--target wasm-wasi`, `--target macos-arm64`.

---

### run

```sh
taupkg run [--release] [-- <args>]
```

Build then immediately run the binary. Arguments after `--` are passed through.

```sh
taupkg run -- --port 8080
```

---

### test

```sh
taupkg test [--verbose]
```

Compile and run every `tests/*.tr` file (includes `[dev-deps]`). A file passes
iff it compiles, runs, and its output contains no `FAILED`. `TAURARO_PATH` is set
automatically so tests can import the project's dependencies.

```
Running 2 test file(s)...
  ok              test_core.tr
  FAIL            test_edge.tr
Test files: 2, passed: 1, failed: 1
```

---

### tree

```sh
taupkg tree
```

Print the resolved dependency graph as a tree.

```
app 0.1.0
`-- liba 1.0.0
    `-- libb 1.0.0
```

---

### audit

```sh
taupkg audit
```

Runs three checks and exits non-zero if any fails. See [Security](#security):

1. **integrity** — re-hash every installed package vs. the `taupkg.lock` checksum,
2. **advisories** — match installed versions against the advisory database,
3. **signatures** — verify any package shipping a `taupkg.sig` against your
   trusted keys (valid+trusted ✓, valid-but-untrusted `~` warning, invalid `!`).

```
Auditing 2 locked package(s)...
OK: integrity verified -- all installed packages match the lock.

Checking 2 package(s) against 1 advisory(ies)...
OK: no known vulnerabilities.

  ✓ shared@1.5.0 signature valid (trusted key)
Signatures: 1 signed, 0 invalid, 0 untrusted.
```

---

### sbom

```sh
taupkg sbom [-o <file>]
```

Emit a CycloneDX 1.5 SBOM of the resolved dependency graph (to stdout, or to a
file with `-o`). See [Security](#security).

---

### env

```sh
taupkg env
```

Print the `TAURARO_PATH` line for compiling this project with `tauraroc`
directly. Shell-appropriate so it can be eval'd on POSIX:

```sh
eval "$(taupkg env)"
tauraroc src/main.tr -o myapp
```

On Windows it prints both a `set` line and a PowerShell `$env:` line.

---

### bindgen

```sh
taupkg bindgen <header> [flags...] [--name <pkg>] [--out <dir>]
```

Generate Tauraro FFI bindings for a C/C++ header **as a ready-to-import taupkg
package**. See [Generating FFI bindings](#generating-ffi-bindings-bindgen).

---

### list

```sh
taupkg list
```

Print all packages currently recorded in `taupkg.lock`.

---

### info

```sh
taupkg info <package>
```

Show detailed information about an installed package (version, source, checksum).

---

### search

```sh
taupkg search <query>
```

Search the official Tauraro package registry.

---

### login

```sh
taupkg login <token>
```

Store a registry token in `~/.taupkg/credentials` for publishing. (You can also
set `TAUPKG_TOKEN` in the environment instead.)

---

### key / sign / verify

```sh
taupkg key gen [--name <id>]      # generate an ed25519 keypair (default: "default")
taupkg key show [--name <id>]     # print the public key
taupkg key trust <pubkey> [--name <label>]   # trust a publisher's public key
taupkg sign <message> [--name <id>]          # print a detached signature
taupkg verify <message> <signature> <pubkey> # VALID / INVALID (exit 0/1)
```

Ed25519 (RFC 8032) signing. The private key is a 32-byte seed stored at
`~/.taupkg/keys/<id>.key` — keep it secret. See [Security](#security).

---

### publish

```sh
taupkg publish [--verbose]
```

Create a source tarball of your project's `src/` directory and publish it to the
registry. Requires a token (via `taupkg login` or `TAUPKG_TOKEN`).

> **Note**: `publish` creates a `.tar.gz` of `src/` — not the compiled binary.

---

### clean

```sh
taupkg clean
```

Delete the `.taupkg/build/` directory. Does not touch downloaded packages in
`.taupkg/packages/`.

---

### install-tauraro

```sh
taupkg install-tauraro [--version x.y.z] [--mirror] [--verbose]
```

Download and install the Tauraro compiler from GitHub releases (or the
`tauraro.org` mirror) for the current platform.

| Flag | Description |
|------|-------------|
| `--version x.y.z` | Install a specific version (default: latest) |
| `--mirror` | Use `tauraro.org` mirror instead of GitHub |

The compiler is installed to `~/.taupkg/bin/tauraroc-{os}-{arch}/` and added to
your `PATH`. Supported platforms: linux-x64, linux-arm64, windows-x64, macos-arm64.

---

## Dependency sources

A dependency's source comes from its inline table, or (for a bare version) from
`[sources]`, or defaults to the registry.

### Inline-table forms (recommended)

```toml
[deps]
a = { path = "../a" }                                   # local path
b = { git = "https://host/b.git", tag = "v1.2.3" }      # git (tag/branch/rev)
c = { github = "user/c", branch = "main" }              # github shorthand
d = { version = "^1.0" }                                 # registry
```

### String / `[sources]` forms

```toml
[deps]
mathlib = "^1.2.0"          # registry lookup by constraint

[sources]
mathlib = "github:user/mathlib@v1.2.3"   # override
utils   = "local:../utils"
httplib = "git:https://gitlab.com/org/httplib.git@main"
archive = "https://example.com/pkg-1.0.tar.gz"   # tarball/zip (downloaded)
```

Local packages are copied (not symlinked) into `.taupkg/packages/`. Git sources
are pinned to the resolved commit SHA in the lock.

---

## Version constraints

Version strings follow [Semantic Versioning](https://semver.org). Path/git/github
dependencies have no semver constraint — they unify as `*`.

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

## Generating FFI bindings (bindgen)

`taupkg bindgen` wraps `tauraroc bindgen` and packages the result as a normal,
ready-to-import taupkg package — then adds guidance on top.

```sh
taupkg bindgen <header> [flags...] [--name <pkg>] [--out <dir>]
```

Examples:

```sh
taupkg bindgen zlib.h --pkg zlib          # C header, pkg-config auto-discovery
taupkg bindgen mylib.hpp -h cpp --macros  # C++ header + function-like macros
taupkg bindgen vendor/api.h -Ivendor      # local header with an include dir
```

### What it produces

A complete package under `<out>/<name>/` (name defaults to the header's
filename):

```
mylib/
├── taupkg.toml     # [package] + [bindgen] provenance (header, mode, includes, regen command)
├── src/
│   ├── mylib.tr    # the generated bindings  (import: `from mylib import ...`)
│   └── mylib_shim.cpp   # C++→C shim, when `-h cpp` (auto-compiled + linked)
├── guide.txt       # how to import, build, and link — tailored to this header
└── README.md
```

Then depend on it like any package and import it (auto-`TAURARO_PATH` finds it):

```toml
[deps]
mylib = { path = "./mylib" }
```

```tauraro
from mylib import some_function
```

### Flags

All `tauraroc bindgen` flags are forwarded verbatim:

| Flag | Description |
|------|-------------|
| `-h cpp` | Bind a C++ header (auto C++→C shim via libclang) |
| `--macros` | Also bind function-like C macros (compile-verified shims) |
| `--pkg <name>` | Auto-discover `-I`/`-D`/`-l` via `pkg-config` |
| `-I<dir>` `-D<macro>` `-std=…` `-isystem <dir>` | Extra clang flags |
| `--cc <compiler>` | Compiler for parsing/shims (e.g. `clang++`) |
| `--name <pkg>` | Package name (taupkg-level; default: from header filename) |
| `--out <dir>` | Where to create the package (taupkg-level; default: cwd) |

### Smart extras

- **`#include` scan** — reports the header's system `<…>` and local `"…"`
  includes so you know its dependencies, with likely link flags for well-known
  libraries (zlib → `-lz`, sqlite3 → `-lsqlite3`, …).
- **Pragma-aware guidance** — reads the generated `# tauraro-cpp-*` link pragmas
  and tells you exactly what links automatically vs. what needs manual flags.
- **Self-validation** — runs `tauraroc --check` on the bindings and reports
  whether they parse and type-check.
- **Symbol inventory + sample import** and a **reproducible regen command**
  recorded in `taupkg.toml [bindgen]` and printed in `guide.txt`.

### Linking the underlying library

- **System libraries** (e.g. via `--pkg`) record link flags as pragmas in the
  bindings, so `taupkg build` links them automatically — no manual `-l`.
- **C++ headers** ship a shim that is auto-compiled and linked (zero-cost — LTO
  inlines the wrapper away).
- **A hand-vendored C source** dropped next to the bindings links automatically
  too: add a `# tauraro-cpp-shim: <file>.c` line to the bindings and the compiler
  builds it with the C driver (unmangled C ABI). See the generated `guide.txt`.

---

## Project layout

```
myproject/
├── taupkg.toml          # manifest
├── taupkg.lock          # lock file (commit this)
├── src/
│   └── main.tr          # entry point (compiled in place)
├── tests/
│   └── test_core.tr     # test files (test_*.tr)
└── .taupkg/
    ├── packages/        # downloaded/copied deps (gitignore this)
    └── build/           # only used by `--sandbox` builds (gitignore this)
```

Add `.taupkg/` to `.gitignore`. Commit `taupkg.lock`.

---

## Building from source

Prerequisites: a working `tauraroc` binary and `gcc`.

```sh
git clone https://github.com/tauraro/taupkg
cd taupkg

# The simplest path: let a recent tauraroc build it directly.
tauraroc src/main.tr -o taupkg
```

If your `tauraroc` predates single-command binary output, emit C and compile it
(see the module list under `src/build/` after `tauraroc src/main.tr --emit c`).

---

## License

MIT — see [LICENSE](LICENSE) for details.
