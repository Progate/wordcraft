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

## License

As the rest of WordCraft (MIT OR Apache-2.0). `browseros/crates/parking_lot_core` keeps its own license
(MIT OR Apache-2.0, see its `LICENSE-*`).
