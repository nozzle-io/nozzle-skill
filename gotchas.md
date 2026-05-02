# Nozzle Gotchas

## §1. No Designated Initializers on MSVC

MSVC C++17 does not support designated initializers (` .member = value`). All descriptor structs must use explicit assignment:

```cpp
// WRONG — does not compile on MSVC
nozzle::sender_desc desc{
    .name = "my-sender",
    .application_name = "MyApp"
};

// CORRECT
nozzle::sender_desc desc;
desc.name = "my-sender";
desc.application_name = "MyApp";
desc.ring_buffer_size = 3;
```

This applies to **all** descriptor structs: `sender_desc`, `receiver_desc`, `texture_desc`, `acquire_desc`, `metal::device_desc`, `metal::texture_wrap_desc`, `d3d11::DeviceDesc`, `d3d11::TextureWrapDesc`.

## §2. C++ API Not Stable for Wrappers

All wrappers (py.nozzle, nozzle.rs, jit.nozzle, etc.) use the **C API** (`nozzle_c.h`). The C++ API may change between versions. If writing a wrapper, bind to the C API only.

## §3. Move-Only Types

`sender`, `receiver`, `frame`, `writable_frame`, `texture`, and `device` are move-only. Attempting to copy produces a compile error:

```cpp
auto snd = nozzle::sender::create(desc).value();
// auto snd2 = snd;           // ERROR: deleted copy constructor
auto snd2 = std::move(snd);  // OK — snd is now invalid
```

After a move, `valid()` returns `false` on the source. Check `valid()` before use.

## §4. Result<T> Must Be Checked

`Result<T>` does not throw. Accessing `.value()` on a failed result is undefined behavior:

```cpp
auto frm = snd.acquire_writable_frame(desc);
// frm.value();  // UB if !frm.ok() — no exception, no assert in release
if (!frm.ok()) {
    // handle frm.error().code and frm.error().message
    return;
}
auto &writable = frm.value();
```

For `Result<void>`, check `.ok()` only — there is no `.value()`.

## §5. sender_info vs connected_sender_info

`sender_info` (from `enumerate_senders()`) contains **only**:
- `name`, `application_name`, `id`, `backend`

It does **NOT** contain `width`, `height`, or `format`. These exist only on `connected_sender_info`, which you get from `receiver.connected_info()` **after** connecting:

```cpp
auto senders = nozzle::enumerate_senders();
for (auto &s : senders) {
    // s has no width/height — can't check dimensions here
}

auto recv = nozzle::receiver::create(recv_desc).value();
// After connection:
auto info = recv.connected_info();
printf("%ux%u %s\n", info.width, info.height, to_string(info.format));
```

## §6. Pixel Row Bytes May Have Padding

`mapped_pixels::row_bytes` is **not** guaranteed to equal `width * bytes_per_pixel`. The GPU may pad rows for alignment. Always use `row_bytes` to advance rows:

```cpp
auto mapped = nozzle::lock_frame_pixels(frame).value();
auto *row = static_cast<uint8_t *>(mapped.data);
for (uint32_t y = 0; y < mapped.height; ++y) {
    process_row(row, mapped.width);
    row += mapped.row_bytes;  // NOT width * bpp
}
nozzle::unlock_frame_pixels(frame);
```

## §7. Backend Headers Require Platform Guards

Backend-specific headers only exist on their target platform. Always guard:

```cpp
#if NOZZLE_HAS_METAL
#include <nozzle/backends/metal.hpp>
#endif

#if NOZZLE_HAS_D3D11
#include <nozzle/backends/d3d11.hpp>
#endif

#if NOZZLE_HAS_OPENGL
#include <nozzle/backends/opengl.hpp>
#endif
```

Including `metal.hpp` on Windows or `d3d11.hpp` on macOS is a compile error. `opengl.hpp` requires `NOZZLE_BUILD_OPENGL=ON` at CMake configure time.

## §8. OpenGL Interop is Copy-Based

The OpenGL interop path (`nozzle::gl::publish_gl_texture`, `nozzle::gl::copy_frame_to_gl_texture`) is **not zero-copy**. It performs a GPU→CPU→GPU roundtrip:

- **macOS**: GL texture → read pixels → write to IOSurface-backed Metal texture
- **Windows**: GL texture → read pixels → write to D3D11 shared texture

This is intentional — GL does not share address space with Metal or D3D11 on current OS versions. Performance is acceptable for moderate resolutions but unsuitable for 4K@60fps GL paths.

**Linux GL interop**: Not implemented. Linux uses DMA-BUF for zero-copy sharing.

## §9. Ring Buffer Full Behavior

When the sender's ring buffer is full (all N textures committed but not yet consumed), `acquire_writable_frame()` returns `ErrorCode::Timeout` **immediately** — it does not block or wait.

The default `acquire_desc::timeout_ms` is 0 (no wait). To wait for a frame:

```cpp
nozzle::acquire_desc acq;
acq.timeout_ms = 16;  // Wait up to 16ms (~1 frame at 60fps)
auto frm = recv.acquire_frame(acq);
```

## §10. D3D11 Shared Handle Lifetime

`d3d11::get_shared_handle()` returns a `HANDLE` owned by the nozzle texture. Do **not** call `CloseHandle()` on it. The handle is closed when the texture is destroyed.

## §11. Metal IOSurface Sync is Polling

On macOS, IOSurface synchronization uses `IOSurfaceLock`/`IOSurfaceUnlock` with polling (500µs sleep between attempts). This is adequate for most use cases but introduces up to 500µs latency on the read path. There is no hardware fence mechanism for cross-process IOSurface access.

## §12. D3D11 Keyed Mutex Uses INFINITE Timeout

The sender side uses `AcquireSync(0, INFINITE)` on the keyed mutex when committing frames. If the receiver crashes while holding the mutex, the sender thread will deadlock. This is a known limitation — crash cleanup relies on the receiver detecting a dead sender on next access failure.

## §13. Linux DMA-BUF Hardcodes renderD128

The Linux backend currently hardcodes `/dev/dri/renderD128` as the GPU device. This works on single-GPU systems but will fail on multi-GPU configurations. A future version will add device enumeration.

## §14. clone_to_owned_texture Not Implemented

`frame::clone_to_owned_texture(device &dev)` exists in the API but returns `ErrorCode::Unknown` in the current version. Do not use it.

## §15. Format Fallback is Sender-Side Only

When `sender_desc::allow_format_fallback` is true (default), the sender may silently use a different format than requested if the platform does not support the requested one. The receiver sees the actual format via `connected_sender_info::format`.

This is **not** about cross-vendor or cross-OS scenarios (nozzle is same-machine only). It handles cases like requesting `rgba8_srgb` on a platform that only supports `rgba8_unorm`.
