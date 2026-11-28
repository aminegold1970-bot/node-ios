# Full JIT on pre-APRR devices (A9/A10) — diagnosis & fix

The Node 24.21 / V8 13.6 build in this repo ships **full JIT (TurboFan) and WebAssembly** on the A9.
This document records why the stock iOS build crashes and how the patch works.

## Symptom

Without the patch, default (JIT) mode crashes immediately (`exit 138`). Under `lldb`:

```
stop reason = EXC_BAD_ACCESS (code=2, address=0x106088960)   # code=2 = protection failure
frame #0: 0x106088960:  stp x29, x30, [sp, #-0x10]!          # a function prologue
```

V8 jumped to code it generated, but the page **isn't executable**. Not a codegen bug — a
memory-permission bug: the code page is read-write, never read-execute.

## Root cause

V8 13.6's `src/base/build_config.h` has three W^X mechanisms:

- `V8_HAS_PTHREAD_JIT_WRITE_PROTECT` — Apple **APRR** (`pthread_jit_write_protect_np`). arm64 macOS / iOS simulator only.
- `V8_HAS_BECORE_JIT_WRITE_PROTECT` — BrowserEngineCore, **iOS 17.4+** only.
- `V8_HAS_PKU_JIT_WRITE_PROTECT` — Intel **PKU**. x86 Linux only.

On a **real iOS 15 arm64 device all three are 0**. The **A9/A10 have no APRR** (that's A11+),
there's no PKU on ARM, and BECORE requires iOS 17.4+. With no mechanism enabled, V8's
`RwxMemoryWriteScope` is a **no-op** and V8 assumes code memory can be permanently RWX. iOS
enforces **W^X** — it won't grant a page both write and execute — so an RWX request comes back
read-write with **execute silently dropped**. Every JIT'd call then faults. V8-on-iOS was only
ever designed for **jitless** (that's how Chrome for iOS ships), so there is no `mprotect` fallback.

The device *is* capable of JIT: `probe/` proves the classic W^X sequence
(`mmap(RW)` → write → `mprotect(RX)` → execute) works with Dopamine's `dynamic-codesigning`.
V8 just doesn't use it by default.

## Implemented fix

All deltas live in [`scripts/ios-source-fixups.sh`](../scripts/ios-source-fixups.sh), applied
after cloning Node and before `configure` — no downstream Node fork:

1. `deps/v8/src/base/build_config.h`: enable `V8_HAS_PTHREAD_JIT_WRITE_PROTECT` for
   `V8_OS_IOS` too. This activates V8's existing write-scope machinery. The new BECORE path is
   left alone (off at min iOS 15; the pthread branch wins in `code-memory-access-inl.h` anyway).
2. `deps/v8/src/base/platform/platform-darwin.cc`: replace the no-op
   `pthread_jit_write_protect_np` flip on iOS with `mprotect(RW)` / `mprotect(RX)` over every
   registered code range (`RegisterJitRange`, declared in `platform.h`).
3. `deps/v8/src/heap/code-range.cc`: register the bounded JS code cage. V8 13.6 already skips
   its macOS-only RWX precommit block on iOS, so code pages start RW.
4. `deps/v8/src/wasm/wasm-code-manager.cc`: `WasmCodeManager::Commit()` still requests
   `kReadWriteExecute`; on iOS commit `kReadWrite` and register the WASM region so the same
   flip makes it executable. WASM has its own code space that the JS cage does not cover.

Run with `./node --predictable --single-threaded` (see [`jit-test.mjs`](../jit-test.mjs)):
the mprotect flip is process-wide, so it must not race a background compiler thread.
`--jitless` remains available as a safe fallback but disables WebAssembly.

## Caveats

- `build_config.h` is included across V8, so changing it invalidates the ccache for nearly all
  V8 translation units (a cold build is ~80 min).
- Covering every code-write path matters: the JS code range and all WASM regions must be
  registered, or the first call into an uncovered page faults.
