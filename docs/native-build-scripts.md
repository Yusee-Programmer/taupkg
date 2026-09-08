# Native (C/C++) dependencies: build scripts

How a taupkg package that wraps a C/C++ library (fetch its source, compile
it, tell tauraroc how to link it) gets that done automatically on
`taupkg build`/`install` — no manual `--link`, no consumer needing the
library pre-installed, and no CI required.

---

## What Cargo does (the model this follows)

Rust's ecosystem solved this exact problem years ago; taupkg's design below
is a deliberately close read of it, adapted to Tauraro's simpler single-
version unity build.

- **`build.rs`**: an optional Rust file at a crate's root. Cargo compiles
  and runs it *before* compiling the crate proper. It's a normal program —
  it can shell out, download files, invoke a C compiler, probe
  `pkg-config`, whatever the library needs.
- **The `cargo:` stdout protocol**: build.rs talks back to Cargo by
  printing lines like `cargo:rustc-link-lib=static=pq` and
  `cargo:rustc-link-search=native=/path`. Cargo folds these into rustc's
  invocation. `cargo:rerun-if-changed=PATH` tells Cargo when to invalidate
  the cached build.rs output.
- **`links = "pq"`** in `Cargo.toml`: declares which native library a
  crate's build script links. Cargo hard-errors if two crates in the same
  dependency graph declare the same `links` value — two static copies of
  the same library would define the same C symbols twice.
- **The `-sys` crate convention**: `libpq-sys`/`openssl-sys`/`libsqlite3-sys`
  are thin crates that are *just* the build script + raw FFI declarations.
  Normal, ergonomic crates (`postgres`, `openssl`, `rusqlite`) depend on the
  `-sys` crate for the low-level binding and wrap it. This maps directly
  onto the `raw.tr` (FFI) vs. everything-else (ergonomic API) split that
  vendored-C taupkg packages already use.
- **`bundled`/`vendored` Cargo feature flags**: `libsqlite3-sys` has a
  `bundled` feature that vendors SQLite's C source and compiles it via the
  `cc` crate instead of linking a system copy; `openssl-sys` has `vendored`,
  which downloads and builds OpenSSL from source via the `openssl-src`
  helper crate. The *consumer* chooses per-project, via a feature flag, not
  the package author choosing once for everyone.
- **Helper crates that keep individual build.rs files small**: `cc` (invoke
  the C/C++ compiler portably, cross-compile-aware), `pkg-config` (wrap
  `pkg-config` discovery), `cmake` (invoke CMake), `vcpkg` (probe a local
  vcpkg install — the standard way Windows Rust crates get a prebuilt
  native library without a from-source build at all).
- **Where the C *source* actually lives**: notably, Cargo does **not** have
  a manifest field for "download this URL." `openssl-src`/similar helper
  crates instead vendor the C source **as data inside a normal, published
  crate** (fetched from crates.io like any dependency, checksummed by the
  registry like any dependency). The generic problem "fetch an arbitrary
  URL as part of resolving a dependency" is solved once, by the existing
  package-fetch mechanism — not re-solved per-library in the manifest
  schema.

## The taupkg design

Two manifest keys plus one execution phase — deliberately not a generic
"describe any C build system in TOML" schema (that's a rabbit hole; see
"What this deliberately does NOT do" below).

### `[package] build = "build.tr"`

A normal Tauraro program at that path. Before `taupkg build` (or `install`,
or `test`) compiles anything, taupkg:

1. Compiles it standalone (`tauraroc build.tr -o <cache>/_build_script`).
2. Runs it with **cwd = the package's own directory** and three env vars
   set: `TAUPKG_OUT_DIR` (a stable, per-package cache directory the script
   owns — downloads/builds go here), `TAUPKG_PKG_NAME`, `TAUPKG_PKG_DIR`.
3. Caches: skipped on the next run if neither the script's own source nor
   the package's lock checksum changed (`~/.taupkg/native/<pkg>-<checksum
   prefix>/.build_script_cache`).

### What the script is expected to *do*

Unlike Cargo, there's no `taupkg:`-prefixed stdout protocol (yet — see
"Deferred" below). A build script's job is simply to leave its **own**
package in a state tauraroc's *existing* auto-link pragma mechanism already
understands: writing/patching a `# tauraro-cpp-linkflags:` (or `-shim:`)
line into one of its own `.tr` modules once it has determined the absolute
paths on this machine. taupkg does not invent a second linking mechanism or
touch how tauraroc builds the final link line — it only guarantees the
script has already run, successfully, before compilation starts. (This is
literally what `scripts/regen-bindings.*` scripts already did by hand in
early vendored-C packages; a build script just means taupkg runs that step
automatically instead of a human remembering to.)

Fetching source is not a new primitive either: a build script can shell out
to `curl`/`tar` exactly the way taupkg's own installer does for `archive:`
dependencies — or, more idiomatically, the C library's source can be
declared as an ordinary `[build-deps]` entry with an `archive:`/`github:`
source (already-existing, zero-new-code taupkg functionality), and the
build script's job shrinks to just "compile what's already been fetched
into `.taupkg/packages/<build-dep-name>/`, then write the pragma."

### `[package] links = "pq"`

At most one package in the *whole resolved graph* may declare a given
`links` value. Checked once, up front (`check_links_conflicts` in
`src/buildscript.tr`), across every resolved package's own `taupkg.toml` —
a clear, early error instead of a confusing duplicate-symbol link failure
much later.

## What this deliberately does NOT do

- **No generic "describe a C build system in TOML" schema.** Real C/C++
  projects use configure/make, Meson, CMake, hand-rolled Makefiles, or
  nothing at all — trying to describe all of those declaratively is how
  you end up reimplementing Autotools. A build script is just a normal
  Tauraro program with `Process`/`File` available; it can shell out to
  whatever the library actually needs.
- **No `taupkg:`-prefixed stdout protocol yet.** Cargo's version exists
  because rustc's own link-line construction lives inside `cargo`, so
  build.rs has to hand it structured instructions. taupkg's auto-link
  pragma mechanism already lives inside *tauraroc*, independently of
  taupkg — a build script editing its own `.tr` file achieves the same
  result with no new protocol. Worth revisiting if a real build script
  needs to influence a *different* package's link line, which the current
  design doesn't support.
- **No topological build-script ordering yet.** Scripts run in lock order;
  a script must not assume another package's script already ran. Fine for
  the first real consumer (a single vendored-C package with no native
  transitive deps); revisit if that stops being true.
- **No vcpkg/pkg-config/cmake helper modules yet.** Cargo's equivalents
  (the `cc`/`pkg-config`/`cmake`/`vcpkg` crates) are what keeps individual
  build.rs files short. taupkg build scripts can do the same work directly
  today (they're normal programs); factoring out a shared helper module is
  worth doing once a second real package needs the same steps, not before.

## Status

Implemented: `Manifest.build`/`Manifest.links` fields
(`src/manifest.tr`), `src/buildscript.tr` (`run_build_script`,
`run_build_scripts`, `check_links_conflicts`), wired into `taupkg build`
and `taupkg test` (`src/main.tr`).

Verified against synthetic test packages: a build script compiles and runs
with the right cwd (scoped via a shell `cd &&` prefix, not `OS.chdir` --
that collides with mingw's own libc `chdir()` in codegen) and env vars
(`TAUPKG_OUT_DIR`/`TAUPKG_PKG_NAME`/`TAUPKG_PKG_DIR`); its cache (keyed
inside the package's own directory, not the global native-artifact cache,
so it's correctly invalidated if `taupkg install` ever wipes/reinstalls the
package) skips redundant re-runs; it runs for the *root* project too, not
just as a dependency; `links` collisions are caught with a clear error.

Also exercised for real against `taupostgres`'s own `build.tr` (which
downloads and builds libpq from source) on a real Windows machine: found
and fixed two genuine bugs in *that* script (not in taupkg itself) —
`bash` on PATH resolving to Windows' WSL-launcher stub instead of MSYS2's
real bash, and a nested-double-quoting bug that truncated a
Windows-style path at its first space when passed through `bash -lc
"..."`. Real PostgreSQL source download + extraction + `./configure`
(~140 checks against the actual MSYS2 toolchain) all succeeded; the run
stopped at a genuine `zlib not found` upstream error (this test machine
doesn't have zlib/OpenSSL dev headers installed) — see taupostgres's
`PROPOSAL.md` for the full account. The taupkg-level mechanism itself
(this document) held up throughout; every bug found was in the *consuming*
build script, not in `buildscript.tr`.
