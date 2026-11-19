# WordCraft on BrowserOS

This branch builds WordCraft for [BrowserOS](https://github.com/Progate/packages/tree/main/packages/browser-os)
(`wasm32-wasip1-threads`). The window, input and CPU painting live in `@progate/browser-word`; this fork only
holds what the WordCraft crates need to build and run there.

## Changes

- **`cfg(target_arch = "wasm32")` means the web build.** Every such `cfg` in `crates/` and `apps/` became
  `cfg(all(target_arch = "wasm32", target_os = "unknown"))`. The web build (wasm-bindgen, `js-sys`, the browser's
  storage and clock) stays as it was; on WASI the crates take the native path (`std::fs`, `std::time`, threads),
  as they do on Linux.
- **`sys-locale`'s `js` feature is for the web build only** (`crates/ui-egui/Cargo.toml`). On WASI it calls
  wasm-bindgen imports that don't exist. The interface language comes from `WORDCRAFT_LANGUAGE` there.
- **`parking_lot_core` waits with std's park / unpark on WASI** (`browseros/crates/parking_lot_core`, patched in
  through `[patch.crates-io]`). Upstream's wasm thread parker panics as soon as a lock is contended unless the
  nightly feature is on; the std of `wasm32-wasip1-threads` implements park / unpark with
  `memory.atomic.wait32` / `notify`, so `thread_parker/wasi.rs` waits on that. The rest is crates.io's 0.9.12.
- **The built-in file browser can be the file dialog** (`Services::builtin_file_dialog`). BrowserOS has no system
  dialog; Open, Save As and Insert › Pictures show the built-in browser without the "system dialog isn't
  available" note, and it lists BrowserOS's own file system.
- **The community links can be turned off** (`WordApp::community_links`): the Discord button in the title bar,
  the File sidebar entry, Help › Community Discord, the About dialog's link and the Options checkbox.
- **Noto Sans JP (as Google Fonts ships it) is a Japanese interface font candidate**
  (`opentype/noto/NotoSansJP-Regular.otf`), which `@progate/browser-word` installs under `/usr/share/fonts`.
- **Temporary names don't use the process id on WASI** (`engine/src/io/safe_save.rs`, `proof/src/lib.rs`).
  `std::process::id` panics there; the safe save stays collision-free through `create_new` and its retry.
- **Saving over an existing document doesn't copy its mode on WASI** (`engine/src/io/safe_save.rs`). WASI preview 1
  has no file modes, so `File::set_permissions` (`fchmod`) always fails with ENOSYS and every Ctrl+S on an opened
  file failed with "Function not implemented (os error 52)". New documents never took that path.

- **Pages render on the calling thread on any wasm** (`crates/render/src/lib.rs`, `default_threads`): the render
  thread count is 0 for `target_arch = "wasm32"`, not only for the web. On BrowserOS (`wasm32-browseros-linux-musl`)
  `available_parallelism` reports the host's cores, and every render would start a new `vello_cpu` pool of that many
  threads, each a Web Worker that takes tens of milliseconds to start.
- **`RenderSettings` without `Default`** (`render_settings`): vello_cpu's `RenderSettings::default()` asks
  `available_parallelism` for a thread count on every call, which on Linux reads cgroup files. The renderer builds its
  settings itself, so a frame no longer pays for that only to override it.

## License

As the rest of WordCraft (MIT OR Apache-2.0). `browseros/crates/parking_lot_core` keeps its own license
(MIT OR Apache-2.0, see its `LICENSE-*`).
