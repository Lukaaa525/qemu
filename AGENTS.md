# AGENTS.md

## Cursor Cloud specific instructions

This is the **QEMU** repository (v10.2.50-dev) — a generic open-source machine emulator and virtualizer written primarily in C, with optional Rust and Python components.

### Build system

- QEMU uses **Meson + Ninja**, wrapped by `./configure` and `Makefile`.
- Ubuntu 24.04's system meson (1.3.2) is too old; QEMU requires **meson >= 1.5.0**. A pip-installed meson 1.5.2 is available at `~/.local/bin/meson`. Ensure `$HOME/.local/bin` is on `PATH`.
- Standard build flow:
  ```
  mkdir -p build && cd build
  ../configure --target-list=x86_64-softmmu,x86_64-linux-user --enable-slirp
  make -j$(nproc)
  ```
- The `build/` directory contains all build artifacts. Out-of-tree builds are the norm.

### Running tests

- **Unit tests:** `cd build && meson test --suite unit`
- **QTest (integration):** `cd build && meson test --suite qtest-x86_64`
- **All tests for a target:** `cd build && make check`
- Tests are run via meson inside the `build/` directory.

### Key binaries (in `build/`)

| Binary | Description |
|---|---|
| `qemu-system-x86_64` | Full system emulator |
| `qemu-x86_64` | Linux user-mode emulator |
| `qemu-img` | Disk image manipulation tool |

### Non-obvious caveats

- QEMU must be built **out-of-tree** in a `build/` subdirectory. Never run `./configure` in the source root.
- The configure script creates a Python venv at `build/pyvenv/` automatically. Do not interfere with it.
- There is no hot-reload; after code changes, run `make -j$(nproc)` in `build/` to rebuild (incremental builds are fast).
- To test the system emulator without a guest OS, use QMP protocol: `qemu-system-x86_64 -nographic -nodefaults -qmp stdio` then send JSON commands.
- To test linux-user emulation, compile a **static** binary and run it through `qemu-x86_64`.
- For full dependency list, see `docs/devel/build-environment.rst` and `scripts/ci/setup/ubuntu/ubuntu-2404-aarch64.yaml`.
