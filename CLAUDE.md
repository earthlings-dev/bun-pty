# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

bun-pty is a cross-platform PTY (pseudoterminal) library for Bun. It uses Bun's FFI (`bun:ffi`) to call into a Rust shared library (`rust-pty/`) built on top of `portable-pty` from WezTerm. This is a **Bun-native, Bun-first, Bun-only** library — do not introduce Node.js compatibility shims, `Buffer` usage, or Node-era patterns.

## Commands

```sh
bun run build:rust          # Compile Rust → shared library (release)
bun run build:ts            # Emit .d.ts declarations to dist/
bun run build               # Both of the above

bun run typecheck            # tsc --noEmit (strict mode, zero errors required)
bun test                    # All tests (unit + skips integration by default)
bun run test:unit           # Unit tests only (no FFI/library needed)
bun run test:integration    # Integration tests (needs built Rust library)
                            # Set RUN_INTEGRATION_TESTS=true (already in script)
bun run test:all            # Unit + integration sequentially

bun test src/terminal.test.ts              # Run a single test file
bun test --test-name-pattern "resize"      # Filter tests by name
```

Linux CI uses `cargo zigbuild --target x86_64-unknown-linux-gnu.2.17` for glibc 2.17 compatibility. Local development uses plain `cargo build --release`.

## Architecture

```
TypeScript (src/)                    Rust (rust-pty/src/lib.rs)
─────────────────                    ──────────────────────────
index.ts                             8 FFI exports (C ABI):
  └─ spawn() → Terminal               bun_pty_{spawn,write,read,
                                       resize,kill,get_pid,
terminal.ts                            get_exit_code,close}
  ├─ resolveLibPath() → dlopen()
  ├─ Terminal class (implements IPty)  Pty struct (Arc-wrapped):
  │   ├─ write()  → bun_pty_write     ├─ 3 threads: wait, read, write
  │   ├─ read loop → bun_pty_read     ├─ crossbeam channels (Msg::Data/End)
  │   ├─ resize() → bun_pty_resize    ├─ pending buffer for partial reads
  │   ├─ kill()   → bun_pty_kill      └─ registry: HashMap<u32, Arc<Pty>>
  │   ├─ onData   (string, TextDecoder streaming)
  │   ├─ onBinary (Uint8Array, raw bytes)
  │   └─ onExit   (exit code + signal)
  │
interfaces.ts
  ├─ IPty, IPtyForkOptions, IExitEvent, IDisposable
  └─ EventEmitter<T>
```

**Handle-based FFI bridge:** `bun_pty_spawn` returns an integer handle. All subsequent FFI calls pass this handle. Rust maps handles to `Arc<Pty>` instances via a global registry.

**Read loop:** The TypeScript `Terminal` polls `bun_pty_read` in an async loop. It fires two events per chunk: `onBinary` (raw `Uint8Array` copy via `slice()`) then `onData` (decoded `string` via streaming `TextDecoder`). The read buffer is reused across iterations — `slice()` copies are necessary for `onBinary`.

**TERM injection:** `IPtyForkOptions.name` is injected as `TERM=<name>` in the environment string passed to FFI spawn, unless the caller explicitly sets `TERM` in `opts.env`.

**Command parsing:** TypeScript `shQuote()` wraps arguments in single quotes for safe tokenization. Rust `shell_words::split()` parses the command line back into tokens.

## Conventions

- **Bun-native types only:** Use `Uint8Array` for bytes, `TextEncoder`/`TextDecoder` for encoding. No `Buffer` from `node:buffer`.
- **Strict TypeScript:** `tsconfig.base.json` enables `noUnusedLocals`, `noUnusedParameters`, `noPropertyAccessFromIndexSignature`, `exactOptionalPropertyTypes`, and all other strict flags. Bracket notation required for `process.env['KEY']`.
- **FFI parameter types:** `FFIType.cstring` accepts `Uint8Array` from `encoder.encode()`. `ptr()` from `bun:ffi` accepts any `TypedArray`.
- Library path resolution supports: `BUN_PTY_LIB` env override, `bun compile` embedded binaries (inline ternary for static analysis), and dev/dist fallback paths.
- Rust debug logging: set `BUN_PTY_DEBUG=1`.

## Git Identity

This is an `earthlings-dev` fork. Use:
```sh
git config user.name "takumi.earth"
git config user.email "165885386+takumi-earth@users.noreply.github.com"
```
