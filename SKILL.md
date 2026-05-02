---
name: nozzle-skill
description: Best practices, gotchas, and API patterns for the nozzle GPU texture sharing library. Use when writing, reviewing, or debugging code that uses nozzle (C++ API, C API, or any wrapper).
license: MIT
---

# Nozzle Skill

Cross-platform C/C++17 library for **local inter-process GPU texture sharing**. Alternative to Syphon and Spout.

**Repo**: https://github.com/nozzle-io/nozzle
**Docs**: https://nozzle-io.org
**Namespace**: `nozzle` (C++), `Nozzle*` handles (C API)

## Reference Files

- **`api-reference.md`** — Complete C++ and C API signatures, all types, enums, structs
- **`gotchas.md`** — Platform-specific gotchas, implementation details, edge cases
- **`patterns.md`** — Code examples for common tasks (CPU write, GPU blit, GL interop, wrapping external textures)

## 30-Second Summary

- **Same-machine only**. No network. Sender and receiver on the same computer.
- **Named model**. Sender publishes under a name, receiver connects by name.
- **Ring buffer**. Sender pre-allocates N GPU textures (default 3). Acquire → write → commit.
- **`Result<T>` everywhere**. No exceptions. **Always check `.ok()` before `.value()`.**
- **Move-only types**. `sender`, `receiver`, `frame`, `texture`, `device` — not copyable.

## Top 5 Gotchas

1. **No designated initializers on MSVC** (C++17). Use explicit assignment for all descriptor structs. → `gotchas.md §1`
2. **`sender_info` has no width/height**. Use `receiver.connected_info()` for dimensions. → `gotchas.md §5`
3. **Backend headers need platform guards** (`#if NOZZLE_HAS_METAL` etc). → `gotchas.md §7`
4. **Ring buffer full = immediate `ErrorCode::Timeout`**. Default timeout is 0. → `gotchas.md §9`
5. **OpenGL interop is copy-based**, not zero-copy. Linux GL interop not implemented. → `gotchas.md §8`

## Naming

- C++ library: all `snake_case`
- C API: `Nozzle*` types, `nozzle_*` functions, `NOZZLE_*` macros
- D3D11 backend only: PascalCase (`DeviceDesc`, `TextureWrapDesc`)

## Platform Macros

| Macro | Platform |
|-------|----------|
| `NOZZLE_HAS_METAL` | macOS |
| `NOZZLE_HAS_D3D11` | Windows |
| `NOZZLE_HAS_DMA_BUF` | Linux |
| `NOZZLE_HAS_OPENGL` | When built with `NOZZLE_BUILD_OPENGL=ON` |

## Wrappers

All wrappers use the **C API** (`nozzle_c.h`), not the C++ API.

| Wrapper | Language |
|---------|----------|
| py.nozzle | Python (nanobind) |
| nozzle.rs | Rust (cargo) |
| nozzle.swift | Swift (SPM) |
| ofxNozzle | C++ (openFrameworks) |
| jit.nozzle | C (Max/MSP) |
| nozzle-TOP | C++ (TouchDesigner) |
| obs-nozzle | C (OBS Studio) |
| blender-nozzle | Python (Blender) |
| nozzle-sokol | C/C++ (single header) |
| tcxNozzle | C++ (TrussC) |
