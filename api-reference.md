# Nozzle API Reference

## C++ API

### Error Handling

```cpp
enum class ErrorCode {
    Ok, Unknown, InvalidArgument, UnsupportedBackend, UnsupportedFormat,
    DeviceMismatch, ResourceCreationFailed, SharedHandleFailed,
    SenderNotFound, SenderClosed, Timeout, BackendError, CommandFailed,
};

struct Error {
    ErrorCode code;
    std::string message;
};

template <typename T>
class Result {
    bool ok() const noexcept;
    explicit operator bool() const noexcept;
    const T &value() const noexcept;     // UB if !ok()
    T &value() noexcept;
    const T *operator->() const noexcept;
    T *operator->() noexcept;
    const Error &error() const noexcept;
};
```

### Sender

```cpp
#include <nozzle/nozzle.hpp>

class sender {
    static Result<sender> create(const sender_desc &desc);
    Result<writable_frame> acquire_writable_frame(const texture_desc &desc);
    Result<void> commit_frame(writable_frame &frame);
    Result<void> publish_external_texture(const texture &tex);
    sender_info info() const;
    Result<void> set_metadata(const metadata_list &metadata);
    bool valid() const;
    // Move-only. No copy.
};
```

### Receiver

```cpp
class receiver {
    static Result<receiver> create(const receiver_desc &desc);
    Result<frame> acquire_frame();
    Result<frame> acquire_frame(const acquire_desc &desc);
    connected_sender_info connected_info() const;
    metadata_list sender_metadata() const;
    bool is_connected() const;
    bool valid() const;
    // Move-only. No copy.
};
```

### Frame

```cpp
class frame {
    frame_info info() const;
    const texture &get_texture() const;
    Result<texture> clone_to_owned_texture(device &dev) const; // NOT IMPLEMENTED — returns error
    void release();  // Also called automatically on destruction
    bool valid() const;
    // Move-only.
};

class writable_frame {
    texture &get_texture();
    const texture_desc &desc() const;
    bool valid() const;
    // Move-only. Must be committed via sender.commit_frame().
};
```

### Pixel Access

```cpp
#include <nozzle/pixel_access.hpp>

struct mapped_pixels {
    void *data;
    uint32_t row_bytes;  // May differ from width * channels (alignment)
    uint32_t width;
    uint32_t height;
    texture_format format;
};

Result<mapped_pixels> lock_frame_pixels(const frame &frm);
void unlock_frame_pixels(const frame &frm);
Result<mapped_pixels> lock_writable_pixels(writable_frame &frm);
void unlock_writable_pixels(writable_frame &frm);
```

### Discovery

```cpp
#include <nozzle/discovery.hpp>

std::vector<sender_info> enumerate_senders();
// sender_info: name, application_name, id, backend
// NOTE: NO width/height/format. Use receiver.connected_info() for those.
```

### Device

```cpp
#include <nozzle/device.hpp>

class device {
    static Result<device> default_device();
    bool supports_format(texture_format format, texture_usage usage) const;
    bool supports_native_format(uint32_t native_format, texture_usage usage) const;
    bool valid() const;
};
```

### Texture

```cpp
class texture {
    const texture_desc &desc() const;
    texture_layout layout() const;
    bool valid() const;
    // Move-only.
};
```

### Backend: Metal (macOS)

```cpp
#if NOZZLE_HAS_METAL
#include <nozzle/backends/metal.hpp>

namespace nozzle::metal {
    struct device_desc { mtl_device_handle device; };
    Result<device> wrap_device(const device_desc &desc);

    struct texture_wrap_desc {
        mtl_texture_handle texture;  // MTLTexture*
        surface_handle io_surface;   // IOSurfaceRef
        pixel_format_value format;
        uint32_t width, height;
    };
    Result<texture> wrap_texture(const texture_wrap_desc &desc);

    mtl_texture_handle get_texture(const texture &tex);
    surface_handle get_io_surface(const texture &tex);
}
#endif
```

### Backend: D3D11 (Windows)

```cpp
#if NOZZLE_HAS_D3D11
#include <nozzle/backends/d3d11.hpp>

namespace nozzle::d3d11 {
    // NOTE: PascalCase naming (Windows convention)
    struct DeviceDesc { ID3D11Device *device; ID3D11DeviceContext *context; };
    Result<device> wrap_device(const DeviceDesc &desc);

    struct TextureWrapDesc {
        ID3D11Texture2D *texture;
        uint32_t dxgi_format;
        uint32_t width, height;
    };
    Result<texture> wrap_texture(const TextureWrapDesc &desc);

    ID3D11Texture2D *get_texture(const texture &tex);
    HANDLE get_shared_handle(const texture &tex);
}
#endif
```

### Backend: OpenGL Interop

```cpp
#if NOZZLE_HAS_OPENGL
#include <nozzle/backends/opengl.hpp>

namespace nozzle::gl {
    struct gl_texture_desc {
        uint32_t name;     // GLuint
        uint32_t target;   // GL_TEXTURE_2D = 0x0DE1
        uint32_t width, height;
        texture_format format;
    };
    // Requires active GL context on calling thread
    Result<void> publish_gl_texture(sender &snd, const gl_texture_desc &desc);
    Result<void> copy_frame_to_gl_texture(const frame &frm, const gl_texture_desc &desc);
}
#endif
```

---

## C API

```c
#include <nozzle/nozzle_c.h>

// Opaque handles
typedef struct NozzleSender NozzleSender;
typedef struct NozzleReceiver NozzleReceiver;
typedef struct NozzleFrame NozzleFrame;

// Error handling — functions return NozzleErrorCode
typedef enum {
    NOZZLE_ERROR_OK = 0,
    NOZZLE_ERROR_UNKNOWN,
    NOZZLE_ERROR_INVALID_ARGUMENT,
    NOZZLE_ERROR_UNSUPPORTED_BACKEND,
    NOZZLE_ERROR_UNSUPPORTED_FORMAT,
    NOZZLE_ERROR_DEVICE_MISMATCH,
    NOZZLE_ERROR_RESOURCE_CREATION_FAILED,
    NOZZLE_ERROR_SHARED_HANDLE_FAILED,
    NOZZLE_ERROR_SENDER_NOT_FOUND,
    NOZZLE_ERROR_SENDER_CLOSED,
    NOZZLE_ERROR_TIMEOUT,
    NOZZLE_ERROR_BACKEND_ERROR,
    NOZZLE_ERROR_COMMAND_FAILED,
} NozzleErrorCode;

// Sender
NozzleErrorCode nozzle_sender_create(const NozzleSenderDesc *desc, NozzleSender **sender);
NozzleErrorCode nozzle_sender_acquire_writable_frame(NozzleSender *sender,
    uint32_t width, uint32_t height, NozzleTextureFormat format, NozzleFrame **frame);
NozzleErrorCode nozzle_sender_commit_frame(NozzleSender *sender, NozzleFrame *frame);
NozzleErrorCode nozzle_sender_lock_writable_pixels(NozzleFrame *frame,
    void **pixels, uint32_t *row_bytes);
void nozzle_sender_unlock_writable_pixels(NozzleFrame *frame);
void nozzle_sender_destroy(NozzleSender *sender);

// Receiver
NozzleErrorCode nozzle_receiver_create(const NozzleReceiverDesc *desc, NozzleReceiver **receiver);
NozzleErrorCode nozzle_receiver_acquire_frame(NozzleReceiver *receiver,
    uint64_t timeout_ms, NozzleFrame **frame);
NozzleErrorCode nozzle_receiver_lock_frame_pixels(NozzleFrame *frame,
    void **pixels, uint32_t *row_bytes);
void nozzle_receiver_unlock_frame_pixels(NozzleFrame *frame);
void nozzle_frame_release(NozzleFrame *frame);
void nozzle_receiver_destroy(NozzleReceiver *receiver);

// Discovery
NozzleErrorCode nozzle_enumerate_senders(NozzleSenderInfo **infos, uint32_t *count);
void nozzle_free_sender_infos(NozzleSenderInfo *infos);
```

---

## All Types

### Enums

| Enum | Values |
|------|--------|
| `backend_type` | `unknown`, `d3d11`, `metal`, `opengl`, `dma_buf` |
| `texture_format` | `unknown`, `r8_unorm`, `rg8_unorm`, `rgba8_unorm`, `bgra8_unorm`, `rgba8_srgb`, `bgra8_srgb`, `r16_unorm`, `rg16_unorm`, `rgba16_unorm`, `r16_float`, `rg16_float`, `rgba16_float`, `r32_float`, `rg32_float`, `rgba32_float`, `r32_uint`, `rgba32_uint`, `depth32_float` |
| `transfer_mode` | `unknown`, `zero_copy_shared_texture`, `gpu_copy`, `cpu_copy` |
| `sync_mode` | `none`, `access_guarded`, `gpu_fence_best_effort` |
| `receive_mode` | `latest_only`, `sequential_best_effort` |
| `frame_status` | `new_frame`, `no_new_frame`, `dropped_frames`, `sender_closed`, `error` |
| `texture_usage` | `none`, `shader_read`, `shader_write`, `render_target`, `shared` (bitmask) |

### Structs

| Struct | Key Fields |
|--------|-----------|
| `sender_desc` | `name`, `application_name`, `ring_buffer_size` (3), `metadata`, `allow_format_fallback` (true) |
| `receiver_desc` | `name`, `application_name`, `receive_mode_val` |
| `texture_desc` | `width`, `height`, `format`, `usage` (shader_read \| shared) |
| `acquire_desc` | `timeout_ms` (0 = no wait) |
| `frame_info` | `frame_index`, `timestamp_ns`, `width`, `height`, `format`, `transfer_mode_val`, `sync_mode_val`, `dropped_frame_count` |
| `sender_info` | `name`, `application_name`, `id`, `backend` — **NO dimensions** |
| `connected_sender_info` | sender_info + `width`, `height`, `format`, `estimated_fps`, `frame_counter`, `last_update_time_ns` |
| `mapped_pixels` | `data`, `row_bytes`, `width`, `height`, `format` |
