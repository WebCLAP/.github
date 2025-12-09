# WCLAP: CLAP compiled to WebAssembly

The [CLAP audio plugin format](https://github.com/free-audio/clap) translates straightforwardly to WebAssembly, and this is supported by some of the main CLAP helpers (e.g. [clap-wrapper](https://github.com/free-audio/clap-wrapper)).

The [Signalsmith C++ examples](https://github.com/geraintluff/signalsmith-clap-cpp) are simple demonstrations of this, and include instructions to build either with [WASI SDK](https://github.com/WebAssembly/wasi-sdk) (recommended) or Emscripten.

## Goals

* Portable audio plugins, across native architectures and browsers
* Sandboxed execution (risk-free use, allows one-click installation)
* Based on existing standards/specs

## Technical description

A WCLAP is a CLAP module compiled to WebAssembly module.  It must:

* export `clap_entry` (an integer representing `clap_plugin_entry *`)
* import memory (must be shared) or export memory (may be shared) - if it does both, hosts should ignore the export.
* export exactly one (growable) function table, so the host can add its functions and reference them.
* export `malloc()` or something like it (`void * malloc(size_t)`) - also see the note about `cabi_realloc()` below

WCLAPs have the extension `.wclap`.  On native machines, their default directories / search paths are the same as CLAP, but with an extra W (e.g. on Linux `/usr/lib/wclap` on Linux, `~/Library/Audio/Plug-Ins/WCLAP` on MacOS).

### Bundles (directories / `.tar.gz`)

If the WCLAP is a directory (or when fetched via URL, a `.tar.gz` which expands into a directory) then the actual binary should be `./module.wasm`.

The WCLAP may import WASI, and hosts should provide this where possible.  If the plugin is a directory, that entire directory should be available through the WASI filesystem (and pointed to in `clap_entry.init()`), but not necessarily at the top-level.

### GUI

CLAP v1.2.7 introduced the [draft webview extension](https://github.com/free-audio/clap/blob/main/include/clap/ext/draft/webview.h).  This is the main way WCLAPs can provide a user-interface, since it requires no platform-specific view-handles or headers, but plugins may still implement `clap.gui` as well (with `CLAP_WINDOW_API_WEBVIEW`) to get show/hide updates and provide size information.

## WebAssembly, WASI, and the future

The WCLAP format should improve "for free" in line with both the CLAP and WebAssembly specifications.  However, since the WebAssembly standards/ecosystem are still fairly new, there are some things worth highlighting for those coming from the native CLAP world:

### WASI

If WebAssembly is the architecture, WASI is the platform, providing system calls.  The widely-supported version is v0.1 (a.k.a. `wasi_snapshot_preview1`) which isn't perfect, but good enough to get many things to work.  You probably don't have to deal with this yourself, toolchains provide standard libraries built on top of this (e.g. [wasi-libc](https://github.com/WebAssembly/wasi-libc)).

Having proven that this approach is possible, the WASI folks stopped work on 0.1, and are rewriting everything in a more modular way for 0.2+.  However, due to current wide support, it's reasonable to expect hosts to support `wasi_snapshot_preview1` in future as well, possibly using polyfills/adapters which forward things to 0.2+.

### `wasm32` or `wasm64`

These are the same instruction sets, and the difference is whether the memory can be large enough that it needs 64-bit integers to address it.  The `wasm32` architecture is the most likely, but hosts should still check the integer type of `clap_entry` (and signature of `malloc`) to be sure.

WASI is sketchily defined for `wasm64`.  There's also not really an advantage to `wasm64` unless you're also using 64-bit memory, and support for that isn't universal in WASM engines either.  Given the choice, you should probably build your WCLAPs as `wasm32`.

### Threads and thread-local storage

In CMake terms, WebAssembly modules are all "executables" because there's no dynamic-library specification.  However, in WebAssembly terms they are "reactors", meaning they have no `main()`-like entry-point, and the functions they provide can be called repeatedly.

In WCLAP, these functions can be called from multiple threads.  For some C++ compiler setups, `thread_local` storage doesn't get initialised on these other threads, so that's something to check and work around for now (possibly by avoiding `thread_local` storage).

WCLAPs _may_ be able to start and manage their own threads.  This is done through [wasi-threads](https://github.com/WebAssembly/wasi-threads), which (as with `wasi_snapshot_preview1`) is the current de-facto standard, and hosts will probably provide support even after it's replaced with something else in WASI 0.3+.

WebAssembly threads _currently_ work by having multiple instances of the same module, using the same shared-memory but otherwise being independent.  There are proposed future WebAssembly-architecture features (known as `shared-everything-threads`) which could support an entirely different model.  When these (eventually) become a reasonable baseline for WASM engines, WCLAPs should be able to use it too.  Browser hosts will probably always need to be [cross-origin isolated](https://developer.mozilla.org/en-US/docs/Web/API/Window/crossOriginIsolated) to enable threads.

### Malloc

WCLAPs need to export this (and their function table) so that hosts can reserve a chunk of memory for their own structures, and register host functions inside the WCLAP's sandboxed world.

This isn't just an issue for WCLAPs, and the newer WASI specs (0.2 onwards) specify `cabi_realloc()` as an export for all WASI-based modules.  Hosts should therefore support this as an alternative to `malloc()`.  The type signature is:

```
void * cabi_realloc(void *prevOrNull, size_t prevSize, size_t alignment, size_t newSize);
```

This can be used to allocate (if `prevSize` is 0), resize (including copying data) or deallocate (`newSize` is 0).  These semantics are not guaranteed for `realloc()` on all platforms, but they are for WASI's `cabi_realloc()`.

### Memory sizes

The JavaScript WebAssembly API doesn't provide a way to inspect the size of memory imports.  This means that browser hosts have to guess what an appropriate size is.

As such, if your `wasm32` WCLAPs import memory, I _suggest_ specifying growable shared memory with a maximum size of 2GB (2^31 bytes).  This is the default in Emscripten (because it's still the maximum supported size for some WebAssembly engines), and it's what [wclap-js](https://github.com/WebCLAP/wclap-js) assumes.
