# Nozzle Patterns

Code examples for common tasks. All examples use the C++ API unless noted.

## 1. Basic Sender (CPU Write)

Write pixel data from CPU to a shared texture.

```cpp
#include <nozzle/nozzle.hpp>
#include <nozzle/pixel_access.hpp>

int main() {
    nozzle::sender_desc desc;
    desc.name = "my-sender";
    desc.application_name = "MyApp";

    auto snd = nozzle::sender::create(desc);
    if (!snd.ok()) return 1;

    nozzle::texture_desc tex_desc;
    tex_desc.width = 1920;
    tex_desc.height = 1080;
    tex_desc.format = nozzle::texture_format::rgba8_unorm;

    while (running) {
        auto frm = snd.value().acquire_writable_frame(tex_desc);
        if (!frm.ok()) continue;  // Ring buffer full

        auto mapped = nozzle::lock_writable_pixels(frm.value());
        if (mapped.ok()) {
            fill_pixels(mapped.value());  // Your pixel fill function
            nozzle::unlock_writable_pixels(frm.value());
        }

        auto commit = snd.value().commit_frame(frm.value());
        if (!commit.ok()) {
            // Frame not published — log and continue
        }
    }
}
```

## 2. Basic Receiver (CPU Read)

Receive a frame and read its pixels.

```cpp
#include <nozzle/nozzle.hpp>
#include <nozzle/pixel_access.hpp>

int main() {
    nozzle::receiver_desc desc;
    desc.name = "my-sender";  // Connect to sender with this name

    auto recv = nozzle::receiver::create(desc);
    if (!recv.ok()) return 1;

    while (running) {
        nozzle::acquire_desc acq;
        acq.timeout_ms = 16;  // Wait up to 16ms
        auto frm = recv.value().acquire_frame(acq);
        if (!frm.ok()) {
            if (frm.error().code == nozzle::ErrorCode::Timeout) continue;
            break;  // Sender closed or error
        }

        auto mapped = nozzle::lock_frame_pixels(frm.value());
        if (mapped.ok()) {
            process_pixels(mapped.value());
            nozzle::unlock_frame_pixels(frm.value());
        }

        // Frame released automatically on next acquire or destruction
    }
}
```

## 3. GPU Blit (Metal/macOS)

Publish an existing Metal texture without CPU readback.

```cpp
#if NOZZLE_HAS_METAL
#include <nozzle/nozzle.hpp>
#include <nozzle/backends/metal.hpp>

void publish_metal_texture(nozzle::sender &snd, MTLTexture *mtl_tex,
                           IOSurfaceRef surface, uint32_t w, uint32_t h) {
    nozzle::metal::texture_wrap_desc wrap;
    wrap.texture = mtl_tex;
    wrap.io_surface = surface;
    wrap.format = nozzle::metal::pixel_format_value::bgra8_unorm;
    wrap.width = w;
    wrap.height = h;

    auto tex = nozzle::metal::wrap_texture(wrap);
    if (!tex.ok()) return;

    snd.publish_external_texture(tex.value());
}
#endif
```

## 4. GPU Blit (D3D11/Windows)

Publish an existing D3D11 texture.

```cpp
#if NOZZLE_HAS_D3D11
#include <nozzle/nozzle.hpp>
#include <nozzle/backends/d3d11.hpp>

void publish_d3d11_texture(nozzle::sender &snd, ID3D11Texture2D *d3d_tex,
                           uint32_t w, uint32_t h) {
    nozzle::d3d11::TextureWrapDesc wrap;
    wrap.texture = d3d_tex;
    wrap.dxgi_format = DXGI_FORMAT_B8G8R8A8_UNORM;
    wrap.width = w;
    wrap.height = h;

    auto tex = nozzle::d3d11::wrap_texture(wrap);
    if (!tex.ok()) return;

    snd.publish_external_texture(tex.value());
}
#endif
```

## 5. GPU Read (Metal/macOS)

Receive a frame and get the Metal texture for GPU rendering.

```cpp
#if NOZZLE_HAS_METAL
#include <nozzle/nozzle.hpp>
#include <nozzle/backends/metal.hpp>

void render_received_frame(nozzle::receiver &recv) {
    auto frm = recv.acquire_frame();
    if (!frm.ok()) return;

    auto &tex = frm.value().get_texture();
    MTLTexture *mtl = nozzle::metal::get_texture(tex);
    IOSurfaceRef surface = nozzle::metal::get_io_surface(tex);

    // Use mtl/surface in your Metal render pipeline
    // ...
}
#endif
```

## 6. OpenGL Interop

Publish a GL texture (copy-based, not zero-copy).

```cpp
#if NOZZLE_HAS_OPENGL
#include <nozzle/nozzle.hpp>
#include <nozzle/backends/opengl.hpp>

void publish_gl_texture(nozzle::sender &snd, GLuint gl_name,
                        uint32_t w, uint32_t h) {
    nozzle::gl::gl_texture_desc desc;
    desc.name = gl_name;
    desc.target = 0x0DE1;  // GL_TEXTURE_2D
    desc.width = w;
    desc.height = h;
    desc.format = nozzle::texture_format::rgba8_unorm;

    auto result = nozzle::gl::publish_gl_texture(snd, desc);
    if (!result.ok()) {
        // Copy failed — check result.error()
    }
}

void receive_to_gl_texture(nozzle::receiver &recv, GLuint gl_name,
                           uint32_t w, uint32_t h) {
    auto frm = recv.acquire_frame();
    if (!frm.ok()) return;

    nozzle::gl::gl_texture_desc desc;
    desc.name = gl_name;
    desc.target = 0x0DE1;
    desc.width = w;
    desc.height = h;
    desc.format = nozzle::texture_format::rgba8_unorm;

    nozzle::gl::copy_frame_to_gl_texture(frm.value(), desc);
}
#endif
```

## 7. Discovery (List Available Senders)

```cpp
#include <nozzle/discovery.hpp>

void list_senders() {
    auto senders = nozzle::enumerate_senders();
    for (auto &s : senders) {
        printf("Sender: %s (app: %s, backend: %d)\n",
               s.name.c_str(), s.application_name.c_str(),
               static_cast<int>(s.backend));
        // NOTE: s has NO width/height — connect to get those
    }
}
```

## 8. C API Sender (Wrapper Pattern)

This is the pattern all wrappers follow.

```c
#include <nozzle/nozzle_c.h>

NozzleSender *create_sender(const char *name) {
    NozzleSenderDesc desc;
    desc.name = name;
    desc.application_name = "MyWrapper";
    desc.ring_buffer_size = 3;

    NozzleSender *sender = NULL;
    NozzleErrorCode err = nozzle_sender_create(&desc, &sender);
    if (err != NOZZLE_ERROR_OK) return NULL;
    return sender;
}

int send_frame(NozzleSender *sender, uint32_t w, uint32_t h) {
    NozzleFrame *frame = NULL;
    NozzleErrorCode err = nozzle_sender_acquire_writable_frame(
        sender, w, h, NOZZLE_FORMAT_RGBA8_UNORM, &frame);
    if (err != NOZZLE_ERROR_OK) return -1;

    void *pixels = NULL;
    uint32_t row_bytes = 0;
    err = nozzle_sender_lock_writable_pixels(frame, &pixels, &row_bytes);
    if (err != NOZZLE_ERROR_OK) {
        nozzle_frame_release(frame);
        return -1;
    }

    /* fill pixels... */
    memset(pixels, 0xFF, row_bytes * h);

    nozzle_sender_unlock_writable_pixels(frame);
    nozzle_sender_commit_frame(sender, frame);
    return 0;
}
```

## 9. C API Receiver (Wrapper Pattern)

```c
#include <nozzle/nozzle_c.h>

void receive_loop(NozzleReceiver *receiver) {
    while (running) {
        NozzleFrame *frame = NULL;
        NozzleErrorCode err = nozzle_receiver_acquire_frame(
            receiver, 16, &frame);  /* 16ms timeout */
        if (err != NOZZLE_ERROR_OK) continue;

        void *pixels = NULL;
        uint32_t row_bytes = 0;
        err = nozzle_receiver_lock_frame_pixels(frame, &pixels, &row_bytes);
        if (err == NOZZLE_ERROR_OK) {
            /* process pixels... */
            nozzle_receiver_unlock_frame_pixels(frame);
        }

        nozzle_frame_release(frame);
    }
}
```

## 10. Custom Device (Non-Default GPU)

Wrap an existing GPU device instead of using the auto-detected default.

```cpp
#if NOZZLE_HAS_METAL
#include <nozzle/backends/metal.hpp>

auto dev = nozzle::metal::wrap_device({.device = my_mtl_device});
if (!dev.ok()) return;

nozzle::sender_desc desc;
desc.name = "custom-gpu";
auto snd = nozzle::sender::create(desc);
// Sender now uses the wrapped device
#endif
```

## 11. Metadata

Attach arbitrary key-value metadata to a sender.

```cpp
nozzle::metadata_list meta;
meta.emplace_back("version", "1.0");
meta.emplace_back("source", "camera-1");
snd.set_metadata(meta);
```

Receivers read it via `receiver::sender_metadata()`.

## 12. Thread Safety

`sender` and `receiver` are individually thread-safe. You can call `acquire_writable_frame` / `commit_frame` from multiple threads on the same sender. However, a single `writable_frame` or `frame` must not be used concurrently — protect individual frames, not the sender/receiver.
