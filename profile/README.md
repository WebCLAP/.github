# WCLAP: CLAP compiled to WebAssembly

The [CLAP plugin format](https://github.com/free-audio/clap) translates straightforwardly to WebAssembly, and this is supported by some official CLAP helpers (e.g. [clap-wrapper](https://github.com/free-audio/clap-wrapper)).

The [Signalsmith C++ examples](https://github.com/geraintluff/signalsmith-clap-cpp) are simple demonstrations of this, and include instructions to build either with [WASI SDK](https://github.com/WebAssembly/wasi-sdk) (recommended) or Emscripten.

## Goals

* Portable audio plugins, across native architectures and browsers
* Sandboxed execution (risk-free use, allows one-click installation)
* Based on existing standards/specs

## Technical description

A WCLAP is a CLAP module compiled to WebAssembly module.  It must:

* export `clap_entry` (an integer which is secretly `clap_plugin_entry *`)
* import memory (must be shared) or export memory (may be shared) - if it does both, hosts should ignore the export.
* export exactly one (growable) function table, so the host can add its functions and reference them.
* export `malloc()` or something like it (`void * malloc(size_t)`)

### Bundles (directories / `.tar.gz`)

If the WCLAP is a directory (or when fetched via URL, a `.tar.gz` which expands into a directory) then the actual binary should be `./module.wasm`.

The WCLAP may import WASI, and hosts should provide this where possible.  If the plugin is a directory, that entire directory should be available through the WASI filesystem (and pointed to in `clap_entry.init()`), but not necessarily at the top-level.

## Fiddly details and future plans

WCLAPs are a relatively small variation on the  CLAP specification,

There are also some concerns which are quite WebAssembly-specific concerns

### `wasm32` or `wasm64`

The `wasm32` architecture is the most likely, but hosts should still check the integer type of `clap_entry` (and signature of `malloc`) to be sure.

WASI is sketchily defined for `wasm64`, and there's not really an advantage to `wasm64` unless you're also using 64-bit memory, and support for that isn't universal in WASM engines either.

### Malloc

WCLAPs need to export this (and their function table) so that hosts can reserve a chunk of memory for their own structures, and register host functions inside the WCLAP's sandboxed world.

This is a common issue, and hosts should support `cabi_realloc()` from the upcoming WASI-p2 spec as an alternative to `malloc()`.  The type signature of this is:

```
void * cabi_realloc(void *prevOrNull, size_t prevSize, size_t alignment, size_t newSize);
```

This can be used to allocate (if `prevSize` is 0), resize (including copying data) or deallocate (`newSize` is 0).  These semantics are not guaranteed for `realloc()` on all platforms, but they are for WASI's `cabi_realloc()`.

### Threads and thread-local storage

In CMake terms, WebAssembly modules are all "executables" because there's no dynamic-library specification.  However, in WebAssembly terms they are "reactors", meaning they have no `main()`-like entry-point, and the functions they provide can be called repeatedly.

In WCLAP, these functions can be called from multiple threads.  For some C++ compiler setups, `thread_local` storage doesn't get initialised on thse other threads, so that's something to check and work around for now (possibly by just avoiding `thread_local` storage).

WCLAPs _may_ be able to start and manage their own threads, if the host supports it.  Browser hosts will need to be [cross-origin isolated](https://developer.mozilla.org/en-US/docs/Web/API/Window/crossOriginIsolated) to enable this.
