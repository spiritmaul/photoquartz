# PhotoQuartz: getting started with this fork

PhotoQuartz is a fork of PhotoCraft 0.6.0, a Rust image editor that aims for Photoshop parity.
This page covers running it on your own machine, making a first change, and checking that
nothing broke. The upstream guides it builds on: [AGENTS.md](../../AGENTS.md),
[development.md](../development.md), [contributing.md](../contributing.md).

The feature map, with every area ranked by importance against readiness, is in
[feature-map.html](feature-map.html). Open it in a browser.

## 1. Install the tools (once)

1. Install Rust from <https://rustup.rs> (stable, 1.95 or newer).
2. Install the system libraries for your OS:
   - **macOS:** `xcode-select --install`
   - **Windows:** Visual Studio Build Tools with the *Desktop development with C++* workload
   - **Debian/Ubuntu:**
     `sudo apt install libxkbcommon-dev libwayland-dev libx11-dev libxrandr-dev libxi-dev libgl1-mesa-dev libgtk-3-dev`
   - Fedora, Arch, openSUSE: see [development.md](../development.md#prerequisites).

## 2. Run the app

```sh
git clone https://github.com/spiritmaul/photoquartz
cd photoquartz
cargo run --release -p photocraft                      # opens the desktop app
cargo run --release -p photocraft -- path/to/photo.jpg  # opens a file
```

The first build compiles about 600 crates and takes 15–40 minutes, depending on the machine.
Later builds only recompile what changed. Always use `--release` for the app itself: image
code is very slow in debug builds.

If the window does not open or the canvas is blank, try the CPU renderer:
`PHOTOCRAFT_CPU_CANVAS=1 cargo run --release -p photocraft`. Logs go to the terminal and to
`~/.config/photocraft/logs/photocraft.log` (Linux), `~/Library/Application Support/Photocraft/logs/`
(macOS) or `%APPDATA%\Photocraft\logs\` (Windows).

## 3. Try it without a window (CLI)

The headless CLI runs the same engine. It is the quickest way to check that a change works.

```sh
cargo run --release -p photocraft-cli -- --help
cargo run --release -p photocraft-cli -- commands | less        # every command id
cargo run --release -p photocraft-cli -- info  photo.jpg
cargo run --release -p photocraft-cli -- convert photo.jpg out.png
```

The built binaries are in `target/release/` (`photocraft`, `photocraft-cli`), so after one build
you can run them directly.

## 4. Make a change

Everything the user can do is a **command** in `crates/engine/src/*_cmds.rs`, registered in
`crates/engine/src/commands.rs`. Menus, shortcuts, the CLI and the MCP server all call commands
by id, so a new command shows up everywhere. The UI (`crates/ui-egui`) stays thin.

A typical loop:

```sh
git checkout -b my-change
# edit code
cargo test -p photocraft-engine --lib --tests   # fast: just the crate you touched
cargo run --release -p photocraft               # look at it
```

House rules that the CI enforces (details in [AGENTS.md](../../AGENTS.md)):

- No `unwrap()`, `expect()` or `panic!` outside tests. Bad input returns an error.
- No `unsafe`.
- Every change comes with a test.
- Never assume 8-bit or sRGB pixels.

## 5. Check before you commit

```sh
cargo fmt --all
cargo clippy -p <crate> --all-targets -- -D warnings
cargo test -p <crate>
cargo xtask layers        # crate dependency rules
cargo xtask ci            # the full gate: fmt, clippy, tests, layers, wasm
```

`cargo xtask ci` is slow (a full workspace test). Run it before opening a pull request, and the
per-crate commands while you work.

## 6. Where to start refining

The ranked table in [feature-map.html](feature-map.html) puts these at the top: performance,
stability, Type, Selections and Retouching. They are things everyday users rely on, and they
are under 60% ready. For a first task, pick a small bug in one of those areas, reproduce it
with the CLI or the app, write a failing test, then fix it.
