# Pictor patches

This repo is a **self-contained snapshot of `pyisyntax` 0.1.5** (from the PyPI
sdist, which flattens the vendored `libisyntax` C source inline — unlike upstream
`anibali/pyisyntax`, where libisyntax is a git submodule). It exists so we can
install iSyntax support as a single git dependency that builds and runs correctly
on **aarch64 Linux**, which upstream does not support (no arm64 Linux wheel, and
the vendored C has x86-centric assumptions).

Two changes on top of stock 0.1.5:

### 1. aarch64 Linux build — `isyntax_build/builder.py`
Adds arm-only `extra_compile_args` so the build selects libisyntax's non-x86
code paths (upstream only enables them for Apple/x86):
- `-D__ARM_NEON__=1` — makes `intrinsics.h` include `<arm_neon.h>` for the NEON code
- `-D__arm64=1` — routes the ltalloc spin-wait to `sched_yield` instead of the x86 `pause`
- `-D_bit_scan_forward=__builtin_ctz`, `-D_popcnt32=__builtin_popcount` — map
  x86-only bit intrinsics onto portable GCC builtins

No consumer needs to set `CPPFLAGS`; the flags are applied by the build itself
and only on `platform.machine() in ("aarch64", "arm64")`.

### 2. Thread-safe reads — `isyntax_build/vendor/libisyntax/src/libisyntax.c`
libisyntax keeps a thread-local memory arena (`local_thread_memory`) that is
initialized only for the thread that opens a slide and for libisyntax's own
worker threads. A read issued from any other thread (e.g. a host application's
reader-pool thread) dereferenced a NULL arena and segfaulted. `libisyntax_tile_read`
and `libisyntax_read_region` now lazily initialize the calling thread's arena:

```c
if (local_thread_memory == NULL) {
    init_thread_memory(0, &global_system_info);
}
```

These are candidates to upstream to `anibali/pyisyntax` / `amspath/libisyntax`;
until then this snapshot is the source of truth for our builds.
