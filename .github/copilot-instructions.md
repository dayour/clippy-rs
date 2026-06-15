# Copilot Instructions for windows-rs

## Repository Overview

This is **windows-rs** — the Rust for Windows project. It provides Rust crates for calling Windows APIs, including raw C-style bindings (`windows-sys`), safer COM/WinRT bindings (`windows`), and a growing ecosystem of supporting crates (error handling, registry, strings, threading, canvas, reactor UI, etc.).

The repository is a Cargo workspace. Large portions of the source code (especially in `crates/libs/windows/`, `crates/libs/sys/`, and `crates/libs/targets/`) are **machine-generated** — do not edit them by hand.

---

## Repository Layout

```
Cargo.toml              # Workspace manifest; also declares workspace-wide Clippy lints
rustfmt.toml            # Unix newlines only (newline_style = "Unix")
.cargo/config.toml      # Build flags and target-specific linker workarounds
crates/
  libs/                 # Published crates (windows, windows-sys, windows-core, etc.)
  samples/              # Example programs demonstrating the APIs
  targets/              # Pre-built import libs (.lib files) for each Windows ABI target
  tests/                # Integration/acceptance test crates
  tools/                # Internal code-generation tools (not published)
docs/                   # Markdown documentation
web/                    # Static website (feature search, etc.)
winmd/                  # Windows metadata (.winmd) files used by the bindgen
```

### Key Library Crates (`crates/libs/`)

| Crate | Purpose |
|---|---|
| `windows` | Safer Rust bindings: COM, WinRT, and Win32 |
| `windows-sys` | Raw C-style bindings for Win32 (no-std friendly) |
| `windows-core` | Core COM/WinRT type support |
| `windows-bindgen` | Code generator that reads `.winmd` metadata and emits Rust |
| `windows-metadata` | Low-level ECMA-335 metadata reader |
| `windows-reactor` | Declarative UI library backed by WinUI 3 |
| `windows-canvas` | 2D graphics on Direct2D |
| `windows-registry` | Registry access |
| `windows-result` | Windows `HRESULT`-based error handling |
| `windows-strings` | `HSTRING`, `PCWSTR`, `PCSTR`, etc. |
| `windows-implement` / `windows-interface` | Proc-macro helpers for COM/WinRT impl |
| `riddle` | Windows metadata authoring tool |
| `windows-rdl` | RDL (Rust Definition Language) for metadata |

### Code Generation Tools (`crates/tools/`)

| Tool | Run command | What it regenerates |
|---|---|---|
| `tool_bindings` | `cargo run -p tool_bindings` | All generated bindings in `crates/libs/windows/` and `crates/libs/sys/` |
| `tool_yml` | `cargo run -p tool_yml` | CI workflow YAML files and MSRV steps in `.github/workflows/msrv.yml` |
| `tool_license` | `cargo run -p tool_license` | License header files across crates |
| `tool_workspace` | `cargo run -p tool_workspace` | `Cargo.toml` workspace metadata |
| `tool_gnu` | `cargo run -p tool_gnu -- all` | GNU import `.a` libs in `crates/targets/` |
| `tool_msvc` | `cargo run -p tool_msvc` | MSVC import `.lib` files in `crates/targets/` |

**After running any generation tool, verify no unexpected diff with:**
```bash
git add -N .
git diff --exit-code
```

---

## Building and Testing

### Prerequisites

- **Rust toolchain**: Most work uses `stable`; `clippy` CI uses `nightly`. MSRV is **1.85** for most crates, **1.95** for `windows-canvas`, `windows-reactor`, and `windows-reactor-setup`.
- **Platform**: Most tests require Windows (the CI uses `windows-2025-vs2026` runners). Format and generation checks run on Ubuntu.
- **LLVM/Clang**: Required for `windows-bindgen` tests. Version 18 on Windows, version 20 on Linux. Set `LIBCLANG_PATH` accordingly.
- **mingw-w64**: Required for building GNU import libs (`tool_gnu`).

### Common Commands

```bash
# Format check (run on Linux or Windows with stable toolchain)
cargo fmt --all -- --check

# Clippy (run on Windows with nightly toolchain; -D warnings is enforced)
cargo clippy --all --tests -- -D warnings

# Run all tests on Windows (stable, x86_64)
cargo test --all --target x86_64-pc-windows-msvc \
  --exclude windows_aarch64_gnullvm \
  --exclude windows_aarch64_msvc \
  --exclude windows_i686_gnu \
  --exclude windows_i686_gnullvm \
  --exclude windows_i686_msvc \
  --exclude windows_x86_64_gnu \
  --exclude windows_x86_64_gnullvm \
  --exclude windows_x86_64_msvc

# Run tests for a single crate
cargo test -p windows-result

# Check no_std compatibility
cargo check -p test_no_std

# Miri test for string literal safety (nightly, Windows)
cargo miri test -p test_strings --test literals
```

> **Note on i686**: The 32-bit target has a 2 GB address space limit. Exclude `test_bindgen` on `i686-pc-windows-msvc` to avoid OOM failures.

### Environment Variable

```bash
RUSTFLAGS=-D warnings   # Enforced in all CI jobs — keep warnings clean
```

---

## CI Workflows (`.github/workflows/`)

| Workflow | Runner | Toolchain | What it checks |
|---|---|---|---|
| `clippy.yml` | Windows | nightly | `cargo clippy --all --tests -- -D warnings` |
| `fmt.yml` | Ubuntu | stable | `cargo fmt --all -- --check` |
| `test.yml` | Windows (x64, ARM64); Windows x64 (i686 cross) | stable / nightly | Full `cargo test` |
| `gen.yml` | Ubuntu | stable | Runs all four generation tools and checks for diff |
| `lib.yml` | Windows | stable | Rebuilds import libs; checks diff in `crates/targets/` |
| `cross.yml` | Ubuntu | stable | Cross-compile to GNU/gnullvm targets |
| `msrv.yml` | Windows | per-crate MSRV | `cargo check` at minimum supported versions |
| `no_std.yml` | Windows | stable + nightly | `cargo check -p test_no_std` |
| `miri.yml` | Windows | nightly | `cargo miri test -p test_strings` |

All PR workflows are triggered on `push` to `master` and on any `pull_request`, except for paths under `.github/ISSUE_TEMPLATE/`, `.github/workflows/web.yml`, `crates/libs/rdl/rdl.md`, and `web/`.

---

## Code Style and Conventions

- **Line endings**: Unix (`\n`) everywhere — enforced by `rustfmt.toml`.
- **Formatting**: `rustfmt` with default settings plus `newline_style = "Unix"`.
- **Clippy lints**: A set of `warn`-level lints is declared in the root `Cargo.toml` under `[workspace.lints.clippy]`. Keep code lint-clean.
- **`unsafe`**: Pervasive due to FFI. All unsafe code must have a safety comment explaining the invariants.
- **Generated files**: If a file header says it is generated, run the relevant tool to update it instead of editing by hand.
- **Cargo features**: The `windows` and `windows-sys` crates are feature-gated per-API. Feature lists live in `crates/libs/windows/features.json` and are generated.
- **No-std support**: Many crates are `no_std`-compatible; do not introduce `std`-only dependencies without checking existing `#![no_std]` attribute usage.

---

## Common Pitfalls

1. **Never hand-edit generated bindings.** Files in `crates/libs/windows/src/`, `crates/libs/sys/src/`, and `crates/targets/` are outputs of code-generation tools. Edit the generator or the `.winmd` metadata, then re-run the tool.

2. **After running any test or gen tool, check for unexpected diffs.** CI will fail with `Tests changed code in the repo.` if tests mutate the repository.

3. **Warnings are errors in CI.** `RUSTFLAGS=-D warnings` is set on all test and clippy jobs. Fix all warnings before pushing.

4. **The `crates/targets/baseline` directory is excluded from the workspace.** Do not add it as a workspace member.

5. **The `windows_slim_errors` and `windows_raw_dylib` cfg flags** are declared in `.cargo/config.toml` (commented out by default). They are opt-in and affect ABI/error-reporting behavior.

6. **GCC 15 / mingw-w64 linker regression.** A `--allow-multiple-definition` linker flag is applied for `x86_64-pc-windows-gnu` in `.cargo/config.toml` to work around a mingw-w64 issue.

---

## Contributing

- Every PR must reference an existing GitHub issue (see `.github/pull_request_template.md`).
- External contributors must sign the Microsoft CLA (automated via CLA bot).
- Start with `cargo fmt`, `cargo clippy`, and targeted `cargo test` before opening a PR.
- For changes that affect generated outputs, run the relevant `tool_*` crate and include the regenerated files in the same PR.
