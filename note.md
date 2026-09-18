# taupkg v0.0.3 — Release Notes

**Focus of this release: Android support (both for `taupkg` itself and for
the `tauraroc` it installs), resumable downloads, and a Cargo `build.rs`
equivalent for native C/C++ dependencies.**

> Pre-1.0 policy: same as Tauraro itself, any `0.x` bump may contain
> breaking changes with no deprecation period. Nothing user-facing was
> removed in this release.

---

## ✨ Highlights

- **Android is a first-class platform now**, not just "happens to work
  under Termux." `taupkg` ships its own statically-linked `android-arm64`
  build (CI-built, no dynamic-linker dependency), and `install-tauraro`'s
  platform detection was rewritten twice this month — first to recognize
  Termux specifically, then broadened to recognize *any* Android Linux
  environment (Termux, or a proot-based distro like UserLAnd/Andronix/
  GNURoot hosting a real Ubuntu/Debian/Alpine guest) via OS-level signals
  (`getprop`, `/system/build.prop`, `ANDROID_ROOT`/`ANDROID_DATA`) that
  don't depend on knowing every terminal app's package name in advance.
- **Every large download taupkg does is now resumable.** A dropped
  connection, a cancel, or the whole process being killed no longer means
  starting over — `install-tauraro`'s SDK zip and `archive:`-sourced
  dependency downloads both pick up from where they left off on retry.
- **Native (C/C++) build scripts** — a deliberately close read of Cargo's
  `build.rs` model, adapted to Tauraro's simpler single-version unity
  build: `[package] build = "build.tr"` and `[package] links = "pq"` let a
  vendored-C package compile its native dependency and wire up linking
  automatically on `taupkg build`/`install`/`test`, with no manual
  `--link` and no CI required.
- **`install-tauraro`'s extraction hardened**: switched from
  PowerShell's `Expand-Archive` (slow, crawled file-by-file) to `tar`
  first, and now extracts into a staging directory so stray archive
  siblings can never leak into `~/.taupkg/bin/` alongside the real SDK
  folder.
- **A real, blocking compiler-resolution bug found and fixed**: `main.tr`'s
  own `CliArgs` (the argument-parsing class taupkg has always used
  internally) started silently resolving against Tauraro's new `std.cli`
  module instead of taupkg's own local file, because both were named
  `cli` and the compiler's resolver prefers a stdlib match by bare name.
  This broke taupkg's build entirely against any tauraro compiler built
  after `std.cli` landed — renamed the local module to `cliargs.tr` to
  remove the collision.

---

## 🚀 Added

### Android support
- **`taupkg-android-arm64`**: a new CI job builds taupkg itself as a
  statically-linked aarch64 binary (same reasoning as `tauraroc`'s own
  Android build: no ELF `PT_INTERP`, so no dependency on any particular
  host distro's dynamic-linker path) — the same binary runs under Termux
  or a proot-based distro without caring which one is hosting it.
- **`install-tauraro` Android detection**, added then broadened twice this
  month:
  1. First pass: `is_android_termux()` — checked `$PREFIX` for
     `"com.termux"`, `uname -o`, and the Termux filesystem path.
  2. Broadened to `is_android()` — layered from most-general to
     most-specific so it works regardless of which app hosts the shell:
     `getprop` succeeding (an Android system binary that simply doesn't
     exist on non-Android Linux, checked first because it generalizes
     across every environment), `/system/build.prop` existing,
     `ANDROID_ROOT`/`ANDROID_DATA` env vars, then Termux-specific signals
     (broadened to match any Termux package id, not just `com.termux`) as
     final fallbacks.
  3. Once detected, `install-tauraro` resolves to the statically-linked
     `tauraroc-android-arm64` release build instead of the
     dynamically-linked `linux-arm64` build, which fails under any
     Android Linux environment's sandboxed filesystem layout with
     linker/loader errors.

### Resumable downloads with progress caching
- `download_with_progress()` (and its foreground fallback) now write a
  small sidecar state file recording which URL a partial download belongs
  to, checked before starting: a matching partial (same URL, some bytes
  already on disk) resumes via curl's `-C -` instead of restarting from
  zero. The state file is written *before* curl starts, specifically so a
  mid-download crash still leaves enough on disk to resume from, and is
  cleared on success (or discarded in favor of a clean restart if the
  saved URL doesn't match — e.g. falling back to the `tauraro.org` mirror
  after GitHub failed, or requesting a different version against the same
  destination path).
- Applies to **every** large taupkg download, not just `install-tauraro`:
  `installer.tr`'s `archive:`-sourced dependency downloads (`.zip`/`.tgz`/
  `.tar.gz` URLs) now go through the same `download_with_progress()`
  instead of a bare one-shot curl call, so an interrupted `taupkg add`/
  `install` for an archive dependency resumes too.

### Native (C/C++) build scripts
- **`[package] build = "build.tr"`** — a normal Tauraro program taupkg
  compiles and runs (cwd = the package's own directory, with
  `TAUPKG_OUT_DIR`/`TAUPKG_PKG_NAME`/`TAUPKG_PKG_DIR` env vars set) before
  `build`/`install`/`test` compiles anything else. Cached: skipped on the
  next run if neither the script's own source nor the package's lock
  checksum changed. A script's job is to leave its own package in a state
  `tauraroc`'s *existing* auto-link pragma mechanism already understands
  (writing/patching a `# tauraro-cpp-linkflags:`/`-shim:` line) — taupkg
  doesn't invent a second linking mechanism, it just guarantees the script
  ran successfully first.
- **`[package] links = "pq"`** — at most one package in the whole resolved
  dependency graph may declare a given `links` value, checked once up
  front (`check_links_conflicts`) across every resolved package's own
  manifest — a clear, early error instead of a confusing duplicate-symbol
  link failure much later (mirrors Cargo's identical `links` key).
- Deliberately does **not** add a generic "describe any C build system in
  TOML" schema, a `taupkg:`-prefixed stdout protocol, topological
  build-script ordering, or vcpkg/pkg-config/cmake helper modules — see
  [`docs/native-build-scripts.md`](docs/native-build-scripts.md)'s own
  "What this deliberately does NOT do" section for the reasoning behind
  each.

---

## 🔧 Changed

- **`install-tauraro`'s extraction rewritten**: `tar` is tried first on
  every platform (bsdtar reads `.zip` and ships on Windows 10+/macOS/most
  Linux; far faster than PowerShell's `Expand-Archive`, which crawled the
  zig-bundled archive file-by-file and made installs feel hung), falling
  back to the platform-native `unzip`/`Expand-Archive`. Extraction now
  lands in a staging directory first; only the real SDK folder is moved
  into `~/.taupkg/bin/`, so any stray siblings the archive carries
  (`src/`, `examples/`, `benchmarks/`, a `.sha256`) are discarded instead
  of leaking in. A sweep step also cleans up anything an *older*, buggy
  extraction may have already spilled directly into `bin/`.

---

## 🐛 Fixed

- **A real, blocking module-resolution bug**: `src/main.tr`'s
  `from cli import ... CliArgs` was resolving against Tauraro's new
  `std.cli` module (added to the compiler's stdlib this month) instead of
  taupkg's own local `src/cli.tr`, since both are named `cli` and the
  resolver prefers the stdlib match by bare name. This broke taupkg's
  build entirely against any tauraro compiler built after `std.cli`
  landed. Renamed the local module to `src/cliargs.tr` (its only import
  site was `main.tr`) to remove the collision. Verified with a real build
  (`taupkg version`, `taupkg help`) against a freshly-built tauraro v0.0.9
  compiler.

---

## 📚 Notes & tips

- **Android**: no separate flag or config needed — `taupkg install-tauraro`
  auto-detects Termux or a proot-based distro and installs the matching
  build. If detection somehow misses your specific environment, `uname -s`
  reporting `"Linux"` is the fallback path, same as any other Linux
  install.
- **Interrupted downloads**: just re-run the same command
  (`install-tauraro` or `add`/`install`) — it picks up from where it left
  off automatically as long as the destination path didn't change (a
  different `--version` or falling back to the mirror both correctly
  start fresh instead of corrupting the resume).

## ⚠️ Known limitations

- The `taupkg-android-arm64` and `tauraroc-android-arm64` builds are
  currently only produced as CI *artifacts*, not yet published as
  downloadable assets on the actual GitHub Releases page for either repo
  — `install-tauraro`/a direct download on a real Android device will
  404 against a tagged release until that's wired into each repo's
  `release` job. Flagged, not yet done.
- Native build scripts have no topological ordering guarantee yet (fine
  for a single vendored-C package with no native transitive deps; not yet
  exercised with two such packages in the same graph) and no
  `taupkg:`-prefixed stdout protocol for influencing a *different*
  package's link line — see `docs/native-build-scripts.md`.

---

## ⬆️ Upgrading

- No source changes required for existing `taupkg.toml` manifests. New
  capabilities are opt-in (`[package] build = "build.tr"`, `links = "..."`,
  Android auto-detection).
- If you vendored a workaround for the slow `Expand-Archive`-based
  `install-tauraro` extraction on Windows, it's no longer needed — `tar`
  is used by default now.

_`taupkg version` reports **taupkg 0.0.3**._
