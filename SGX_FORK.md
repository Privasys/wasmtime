# Wasmtime SGX Fork — Rationale

This document explains every change made in the Privasys wasmtime fork
(`sgx` branch) to support running wasmtime inside Intel SGX enclaves via
the [Teaclave SGX SDK](https://github.com/apache/incubator-teaclave-sgx-sdk).

All changes are gated behind `#[cfg(target_vendor = "teaclave")]` so that
non-SGX builds are completely unaffected.

---

## Background

[Enclave OS (Mini)](https://github.com/Privasys/enclave-os-mini) runs
WebAssembly Components inside SGX enclaves using wasmtime's AOT
compilation pipeline. WASM apps are pre-compiled to `.cwasm` on the host,
then loaded into the enclave dynamically over the RA-TLS wire protocol.
The enclave deserializes and executes them.

SGX enclaves are a constrained environment:

- **No system calls** — `mmap`, `mprotect`, `sigaction`, `open`, etc.
  are not available.
- **No process control** — `std::process::abort()` does not exist.
- **Filtered CPUID** — the CPU feature detection instruction returns
  masked results inside SGX.
- **No filesystem** — `/tmp`, `/proc`, `/dev` do not exist.
- **Custom target triple** — the Teaclave SDK compiles to
  `x86_64-unknown-teaclave-sgx`, not `x86_64-unknown-linux-gnu`.

Wasmtime already has a `sys::custom` backend with C-ABI hooks for memory,
traps, and TLS — designed for `no_std` environments. This fork routes the
Teaclave target to that backend and fixes a handful of subsystems that
assume a full POSIX environment.

---

## Commit 1 — Route Teaclave target to `sys::custom`

**Commit:** `16f076a45` — *feat(sgx): route Teaclave target to sys::custom C API*

### `crates/wasmtime/Cargo.toml` — New `sgx` feature + dependency gates

A new `sgx` feature is added:

```toml
sgx = ["runtime", "custom-virtual-memory", "custom-native-signals"]
```

This enables the two `custom-*` features that activate the C API backend.

Two platform-gated dependency blocks are updated to exclude Teaclave:

- **`memfd`** — Linux memory-mapped file descriptors. Excluded because SGX
  has no `/proc/self/fd` or `memfd_create`.
- **`rustix`** — Low-level POSIX bindings. Excluded because system calls
  are not available inside an enclave.

### `crates/wasmtime/build.rs` — Exclude Teaclave from `supported_os`

The build script sets `supported_os = true` for `unix + std`, which
activates the `sys::unix` module. The fix adds `&& !teaclave`:

```rust
let teaclave = cfg_is("target_vendor", "teaclave");
let supported_os = (unix || windows) && cfg!(feature = "std") && !teaclave;
```

Without this, the build would select `sys::unix` and try to compile code
that calls `mmap`, `mprotect`, `sigaction`, etc.

### `crates/wasmtime/src/runtime.rs` — Skip unix extensions

The `cfg_if!` chain that conditionally exposes `pub mod unix` (Unix-specific
extensions like `PoolingAllocationConfig::create_memfd_image`) now has a
Teaclave branch above the `unix` branch:

```rust
} else if #[cfg(target_vendor = "teaclave")] {
    // SGX enclave — no unix extensions (uses sys::custom)
} else if #[cfg(unix)] {
    pub mod unix;
```

### `crates/wasmtime/src/runtime/vm/sys/mod.rs` — Route to `sys::custom`

The core routing change. The `cfg_if!` chain that selects the system
backend now matches Teaclave before `unix`:

```rust
} else if #[cfg(target_vendor = "teaclave")] {
    // SGX enclave — use the custom C API backend.
    mod custom;
    pub use custom::*;
} else if #[cfg(windows)] {
```

This is the fundamental change. The embedding crate (`enclave-os-wasm`)
provides the `extern "C"` symbols that `sys::custom::capi` declares:

| C API function | SGX implementation |
|---|---|
| `wasmtime_mmap_new` | RWX code pool (≤1 MiB) or heap alloc (>1 MiB) |
| `wasmtime_mprotect` | no-op (pool = RWX, heap = RW) |
| `wasmtime_munmap` | heap dealloc (pool pages are retained) |
| `wasmtime_mmap_remap` | zero memory |
| `wasmtime_page_size` | 4096 |
| `wasmtime_init_traps` | `sgx_register_exception_handler` (VEH) |
| `wasmtime_tls_get/set` | `AtomicPtr` (single-threaded per TCS) |
| `wasmtime_memory_image_*` | disabled (no CoW / memfd in SGX) |

---

## Commit 2 — De-activate incompatible subsystems

**Commit:** `46a5a99f` — *fix: More de-activations for Teaclave*

### 1. `crates/wasmtime/src/engine/serialization.rs` — Skip ISA flags and OS triple checks

When loading a pre-compiled `.cwasm`, wasmtime's `check_compatible`
validates that the module's metadata matches the host. Two checks fail in
the SGX cross-compilation scenario:

#### OS triple mismatch

The AOT compiler runs on `x86_64-unknown-linux-gnu` and embeds that triple
in the `.cwasm`. The enclave runtime reports `x86_64-unknown-teaclave-sgx`.
Without the skip, deserialization fails:

> *"Module was compiled for operating system 'linux'"*

This is a **hard blocker** — no AOT module can load without this change.

#### ISA flags mismatch (cf. [wasmtime #3897](https://github.com/bytecodealliance/wasmtime/issues/3897))

When Cranelift compiles a `.cwasm`, it queries CPUID on the host to detect
CPU features (SSE4.2, AVX2, BMI2, POPCNT, etc.) and enables corresponding
code-generation optimizations. These feature flags are embedded in the
`.cwasm` metadata.

At load time, wasmtime calls `check_isa_flags`, which uses
`std::is_x86_feature_detected!()` — internally calling `CPUID` — to
verify the host CPU supports those features.

**The problem:** Intel SGX filters CPUID results. The `CPUID` instruction
inside an enclave doesn't trap to the OS, but the SGX microcode **masks
certain feature bits** depending on the SGX version and enclave attributes.
A feature like AVX2 may be physically present and fully functional, but
SGX-filtered CPUID reports it as absent.

The AOT compiler, running **outside** SGX on the same physical CPU, sees
the real CPUID and enables optimizations. The enclave, running on the
**same CPU**, gets a filtered CPUID and thinks the feature is missing:

> *"compilation settings of module incompatible with native host:
> compilation setting "has_avx2" is enabled, but not available on the
> host"*

**Why skipping is safe:**

1. **Same physical CPU.** The AOT compiler and enclave always run on the
   same machine. CPU identity is verified through attestation (MRENCLAVE /
   MRSIGNER).
2. **Code integrity.** The SHA-256 hash of every loaded WASM app is
   embedded in the RA-TLS certificate (OID `1.3.6.1.4.1.65230.2.3`).
   Nobody can substitute a differently-compiled module without the
   hash changing.
3. **Feature flags are additive.** If Cranelift emitted AVX2
   instructions, the CPU physically supports them. The generated code
   executes correctly — only the CPUID reporting is wrong.

The check exists to prevent loading a `.cwasm` compiled for a more capable
CPU onto a less capable one. In SGX, that scenario is ruled out by the
integrity guarantees above.

Both guards use:

```rust
#[cfg(not(target_vendor = "teaclave"))]
```

Outside SGX (normal Linux/Windows/macOS), the checks still run at full
strictness.

> **Alternative considered:** Providing a custom `detect_host_feature`
> callback via `Config::detect_host_feature()` that returns `Some(true)`
> for all features. This avoids patching wasmtime source but requires
> hardcoding the host's feature set inside the enclave. Since we control
> the compilation pipeline end-to-end and MRENCLAVE guarantees integrity,
> the compile-time `#[cfg]` skip is simpler and equally correct.

### 2. `crates/wasmtime/src/profiling_agent.rs` — Disable perfmap

The `perfmap` profiling agent writes JIT symbol maps to
`/tmp/perf-<pid>.map` for Linux `perf` integration. This requires
`std::fs` and `/tmp` — neither exists in SGX.

```rust
if #[cfg(all(unix, feature = "std", not(target_vendor = "teaclave")))] {
    mod perfmap;
```

Without this, compilation fails because `std::fs::File::create` resolves
to Teaclave's stub.

### 3. `crates/wasmtime/src/runtime/debug.rs` — `abort` → `panic`

The `abort_on_republish_error` function calls `std::process::abort()` when
re-publishing executable code (after GDB breakpoint editing) fails. In
SGX, `abort()` is not available — the enclave does not own the process.

A Teaclave-specific variant uses `panic!()` instead:

```rust
#[cfg(all(feature = "std", target_vendor = "teaclave"))]
fn abort_on_republish_error(e: crate::Error) -> ! {
    panic!("abort: failed to re-publish executable code");
}
```

This code path should never be hit in production (it relates to GDB
breakpoint editing, which doesn't happen inside SGX).

### 4. `crates/wasmtime/src/runtime/store.rs` — `abort` → `panic`

`StoreOpaque` handles invalid trap faults by printing a diagnostic to
stderr and calling `std::process::abort()`. Same issue — `abort()` and
`eprintln!` are not available in SGX.

A Teaclave-specific branch panics with the PC and faulting address:

```rust
if #[cfg(target_vendor = "teaclave")] {
    let _ = pc;
    panic!("wasmtime: invalid fault at pc=0x{:x}, addr=0x{:x}", pc, addr);
}
```

---

## Summary

| File | Commit | Change | Reason |
|------|--------|--------|--------|
| `Cargo.toml` | 1 | `sgx` feature; exclude `memfd`/`rustix` | No `memfd_create` or syscalls in SGX |
| `build.rs` | 1 | Exclude Teaclave from `supported_os` | Prevent `sys::unix` module selection |
| `runtime.rs` | 1 | Skip `pub mod unix` for Teaclave | Unix extensions use unavailable APIs |
| `sys/mod.rs` | 1 | Route Teaclave → `sys::custom` | Core change: use C API backend |
| `serialization.rs` | 2 | Skip OS triple check | Compiler = linux-gnu, enclave = teaclave |
| `serialization.rs` | 2 | Skip ISA flags check | SGX CPUID filtering hides real CPU features |
| `profiling_agent.rs` | 2 | Disable perfmap | No filesystem in SGX |
| `debug.rs` | 2 | `abort` → `panic` | No `std::process::abort` in SGX |
| `store.rs` | 2 | `abort` → `panic` | No `std::process::abort` in SGX |

All changes are conditional on `target_vendor = "teaclave"`. Normal
wasmtime builds are completely unaffected.
