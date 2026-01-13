# QNX Screen screen_post_window() and Qualcomm OpenWF Display Integration

## Overview

This article explains how QNX Screen Graphics Subsystem integrates with hardware vendor implementations, specifically focusing on Qualcomm's OpenWF Display (WFD) architecture. Understanding this flow is crucial for embedded graphics development on QNX platforms.

---

## Architecture Overview

### System Stack

```c
┌─────────────────────────────────────┐
│ Application (SCREEN_POST_WINDOW) │
├─────────────────────────────────────┤
│ QNX Screen Graphics Subsystem │
│ (libscreen.so) │
├─────────────────────────────────────┤
│ Screen Composition Manager │
│ (screen compositor) │
├─────────────────────────────────────┤
│ OpenWF Display (WFD) Layer │ ← Hardware abstraction
├─────────────────────────────────────┤
│ Vendor-Specific WFD Driver │ ← Vendor implements
│ (libwfdqcom.so for Qualcomm) │
├─────────────────────────────────────┤
│ DRM/KMS (Kernel Mode Setting) │
├─────────────────────────────────────┤
│ Kernel Driver (msm_drm) │
├─────────────────────────────────────┤
│ Hardware (MDP + Display HW) │
└─────────────────────────────────────┘
```


### Key Components

| Component | Purpose |
|-----------|---------|
| **Screen Library** | Application-facing API for graphics operations |
| **Screen Compositor** | Manages window composition and layer ordering |
| **OpenWF Display** | Hardware-independent display abstraction (Khronos standard) |
| **Vendor WFD Driver** | Hardware-specific implementation |
| **DRM/KMS** | Linux kernel display framework |

---

## The SCREEN_POST_WINDOW Journey

### Function Signature

```c
int screen_post_window(screen_window_t win, 
                    screen_buffer_t buf, 
                    int count, 
                    const int *dirty_rects, 
                    int flags);
```

## What Happens Step-by-Step


### Application renders to buffer
```
screen_get_window_property_pv(win, SCREEN_PROPERTY_RENDER_BUFFERS, &bufs);
// ... render content to bufs[0] ...

// Post the buffer to make it visible
screen_post_window(win, bufs[0], 1, NULL, 0);
```

### Screen Library Processing

* Validates window and buffer handles
* Queues buffer for composition
* Sends message to Screen compositor process

### Screen Compositor

* Receives posted buffers from multiple windows
* Performs composition based on:
** Z-order (window stacking)
** Transparency/alpha blending
** Clipping regions
* Assigns windows to hardware pipelines

### Dirty Rectangle Optimization
```
// Only update changed regions
int dirty_rects[] = {x, y, width, height};
screen_post_window(win, buf, 1, dirty_rects, 0);
```

| Flag | Behavior |
|-----------|---------|
| **SCREEN_WAIT_IDLE** | Blocks until buffer is displayed |
| **SCREEN_POST_ASYNC** | Returns immediately without waiting |
| **0 (default)** | Standard vsync-synchronized posting |
---

## OpenWF Display Abstraction Layer

### Why OpenWF Display?

OpenWF Display provides a vendor-neutral API for display management, allowing the same application code to run on different hardware platforms.

### Core Concepts

Devices: Physical display hardware units

```
WFDDevice device = wfdCreateDevice(WFD_DEFAULT_DEVICE_ID, NULL);
```

Ports: Physical display outputs (HDMI, LVDS, DSI, DisplayPort)

```
WFDPort port = wfdCreatePort(device, port_id, NULL);
WFDPortMode mode = wfdGetPortMode(device, port);
```

Pipelines: Hardware composition layers (overlays)

```
WFDPipeline pipeline = wfdCreatePipeline(device, pipeline_id, NULL);
```

Sources: Image/buffer sources for pipelines

```
WFDSource source = wfdCreateSourceFromStream(device, pipeline, stream, NULL);
```

### Pipeline Architecture
```
Source Buffer          Pipelines              Display
┌──────────┐          ┌──────────┐
│ Video    │ ────────→│Pipeline 0│─┐
└──────────┘          │ (Video)  │ │
                   └──────────┘ │
┌──────────┐          ┌──────────┐ │
│ UI       │ ────────→│Pipeline 1│─┼─→ Composition → Output
└──────────┘          │(Graphics)│ │
                   └──────────┘ │
┌──────────┐          ┌──────────┐ │
│ Cursor   │ ────────→│Pipeline 2│─┘
└──────────┘          │ (Cursor) │
                   └──────────┘
```


### Standard WFD API Flow
```
// 1. Create device
WFDDevice device = wfdCreateDevice(WFD_DEFAULT_DEVICE_ID, NULL);

// 2. Enumerate and configure port
WFDint port_ids[10];
wfdEnumeratePorts(device, port_ids, 10, NULL);

// 3. Create pipeline (hardware layer)
WFDPipeline pipeline = wfdCreatePipeline(device, pipeline_id, NULL);

// 4. Create source from buffer
WFDSource source = wfdCreateSourceFromStream(device, pipeline, stream, NULL);

// 5. Bind source to pipeline
wfdBindSourceToPipeline(device, pipeline, source, 
                     WFD_TRANSITION_AT_VSYNC, NULL);

// 6. Commit changes to hardware
wfdDeviceCommit(device, WFD_COMMIT_ENTIRE_DEVICE, WFD_INVALID_HANDLE);
```

## Qualcomm Implementation Details in QNX

### Qualcomm Display Architecture

```
┌──────────────────────────────────────────────────┐
│  libwfdqcom.so (Qualcomm WFD for QNX)            │
├──────────────────────────────────────────────────┤
│  QNX Graphics Framework                          │
├──────────────────────────────────────────────────┤
│  Screen Driver Interface                         │
├──────────────────────────────────────────────────┤
│  MDP (Mobile Display Processor) Hardware         │
│  ┌────────────────────────────────────────┐      │
│  │ SSPP (Source Surface Processor Planes) │      │
│  │  ├─ VIG Pipes (Video + Graphics)       │      │
│  │  ├─ RGB Pipes (Graphics only)          │      │
│  │  └─ DMA Pipes (Simple overlays)        │      │
│  ├────────────────────────────────────────┤      │
│  │ Layer Mixer (Blending engine)          │      │
│  ├────────────────────────────────────────┤      │
│  │ DSPP (Display Post-Processor)          │      │
│  │  ├─ Color correction                   │      │
│  │  ├─ Dithering                          │      │
│  │  └─ Sharpening                         │      │
│  ├────────────────────────────────────────┤      │
│  │ Interface (DSI/HDMI/DP)                │      │
│  └────────────────────────────────────────┘      │
└──────────────────────────────────────────────────┘
```

### MDP Pipeline Types


| Pipeline Type | Purpose | Capabilities |
|-----------|---------|---------|
| VIG (Video) | Video playback | Scaling, YUV→RGB, rotation, deinterlacing |
| RGB (Graphics) | UI rendering | Alpha blending, basic transforms |
| DMA (Direct) | Cursors, simple overlays | Minimal processing, low latency |
---

### Qualcomm-Specific WFD Implementation

```
// In libwfdqcom.so for QNX

WFDDevice wfdCreateDevice(WFDint deviceId, const WFDint *attribList) {
 DEVICE *dev = calloc(1, sizeof(DEVICE));
 
 // Open QNX graphics device
 dev->gfx_fd = open("/dev/screen", O_RDWR);
 
 // Initialize MDP resources through QNX interface
 screen_get_display_property_iv(display, SCREEN_PROPERTY_PIPELINE_COUNT, 
                                &dev->num_pipelines);
 
 // Setup memory allocator
 dev->mem_fd = open("/dev/mem", O_RDWR | O_SYNC);
 
 // Initialize hardware pipelines
 init_mdp_pipelines(dev);
 
 return (WFDDevice)dev;
}
```

Qualcomm on QNX uses shared memory and physical memory allocation:

```
// Allocate physically contiguous buffer
void *virt_addr;
off64_t phys_addr;
size_t buffer_size = width * height * 4;  // RGBA

// Map physical memory
virt_addr = mmap64(NULL, buffer_size, 
                PROT_READ | PROT_WRITE | PROT_NOCACHE,
                MAP_SHARED | MAP_PHYS,
                NOFD, 
                phys_addr);

// Or use QNX Screen allocator
screen_create_buffer(&buf);
screen_set_buffer_property_iv(buf, SCREEN_PROPERTY_PHYSICALLY_CONTIGUOUS, 
                           &physically_contiguous);
```

QNX-Specific Integration

```
// Qualcomm WFD on QNX uses direct hardware access

int qcom_wfd_commit(DEVICE *dev) {
 // For each active pipeline
 for (int i = 0; i < dev->num_pipelines; i++) {
     PIPELINE *pipe = &dev->pipelines[i];
     
     if (!pipe->enabled) continue;
     
     SOURCE *src = pipe->staged_source;
     
     // Get physical address from Screen buffer
     off64_t phys_addr;
     screen_get_buffer_property_llv(src->screen_buf,
                                    SCREEN_PROPERTY_PHYSICAL_ADDRESS,
                                    &phys_addr);
     
     // Program MDP registers directly
     configure_mdp_sspp(pipe->hw_pipe_id, phys_addr, 
                       src->format, src->width, src->height);
     
     configure_layer_mixer(pipe->mixer_id, pipe->zorder, 
                          pipe->alpha);
 }
 
 // Trigger hardware update at next vsync
 trigger_mdp_flush();
 
 return 0;
}
```

## Deep Dive: wfdBindSourceToPipeline
```
WFD_API_CALL void WFD_APIENTRY
wfdBindSourceToPipeline(WFDDevice device, 
                     WFDPipeline pipeline, 
                     WFDSource source, 
                     WFDTransition transition,
                     const WFDRect *region) WFD_APIEXIT
```
__WFDDevice device__

Handle to the display device (typically represents one display controller)

__WFDPipeline pipeline__

Hardware layer/overlay to use

```
typedef struct {
 uint32_t hw_pipe_id;     // MDP_SSPP_VIG0, RGB0, etc.
 uint32_t z_order;        // Stacking order (0=bottom)
 WFDRect src_rect;        // Source crop region
 WFDRect dst_rect;        // Destination position/size
 uint32_t alpha;          // Global alpha (0-255)
 bool needs_commit;       // Dirty flag
} PIPELINE;
```

__WFDSource source__:

```
typedef struct {
 void *buffer_handle;     // ION/dmabuf handle
 uint32_t phys_addr;      // Physical memory address
 uint32_t format;         // WFD_FORMAT_RGBA8888, NV12, etc.
 uint32_t width;          // Buffer width
 uint32_t height;         // Buffer height
 uint32_t stride;         // Bytes per row
 WFDEGLImage egl_image;   // Optional EGL integration
} SOURCE;
```

__WFDTransition transition__

| Transition | Behavior |
|-----------|---------|
| WFD_TRANSITION_IMMEDIATE | Apply immediately (may tear) |
| WFD_TRANSITION_AT_VSYNC | Wait for vertical blank |
| Vendor extensions |  	Fade, wipe, or other effects |
---

__WFDRect region : Source crop rectangle (NULL = entire source)__

```
typedef struct {
 WFDint x;         // Left offset
 WFDint y;         // Top offset
 WFDint width;     // Crop width
 WFDint height;    // Crop height
} WFDRect;
```

### Implementation Flow

```
void wfdBindSourceToPipeline(WFDDevice device, 
                          WFDPipeline pipeline, 
                          WFDSource source,
                          WFDTransition transition,
                          const WFDRect *region) {
 
 // 1. Validate handles
 DEVICE *dev = (DEVICE*)device;
 PIPELINE *pipe = (PIPELINE*)pipeline;
 SOURCE *src = (SOURCE*)source;
 
 if (!dev || !pipe || !src) {
     wfdSetError(device, WFD_ERROR_BAD_HANDLE);
     return;
 }
 
 // 2. Extract buffer information
 uint32_t phys_addr = src->phys_addr;
 uint32_t format = src->format;
 uint32_t stride = src->stride;
 
 // 3. Set source crop region
 if (region) {
     pipe->src_x = region->x;
     pipe->src_y = region->y;
     pipe->src_w = region->width;
     pipe->src_h = region->height;
 } else {
     // Use entire source
     pipe->src_x = 0;
     pipe->src_y = 0;
     pipe->src_w = src->width;
     pipe->src_h = src->height;
 }
 
 // 4. Configure MDP SSPP (hardware layer)
 struct mdp_sspp_config config = {
     .base_addr = phys_addr,
     .format = convert_wfd_to_mdp_format(format),
     .width = src->width,
     .height = src->height,
     .stride = stride,
     .src_rect = {pipe->src_x, pipe->src_y, 
                  pipe->src_w, pipe->src_h},
 };
 
 // 5. Apply hardware-specific optimizations
 if (src->format & UBWC_FLAG) {
     // Enable Qualcomm's compression
     config.compression = MDP_COMPRESSION_UBWC;
 }
 
 if (is_yuv_format(format)) {
     // Setup color space conversion
     config.csc_matrix = CSC_BT709_TO_RGB;
 }
 
 // 6. Stage configuration (doesn't apply to hardware yet)
 pipe->staged_config = config;
 pipe->staged_source = src;
 pipe->transition = transition;
 
 // 7. Mark as dirty for commit
 pipe->needs_commit = true;
 dev->pending_changes = true;
 
 // Note: Changes won't be visible until wfdDeviceCommit() is called!
}
```

### What Happens in Hardware

```
// For Qualcomm MDP hardware

// 1. Configure SSPP (Source Surface Processing Pipe)
MDP_SSPP_SRC_SIZE = (height << 16) | width;
MDP_SSPP_SRC_IMG_SIZE = (height << 16) | width;
MDP_SSPP_SRC_XY = (y_offset << 16) | x_offset;
MDP_SSPP_OUT_SIZE = (dst_height << 16) | dst_width;
MDP_SSPP_OUT_XY = (dst_y << 16) | dst_x;

// 2. Configure fetch from memory
MDP_SSPP_SRC0_ADDR = phys_addr;
MDP_SSPP_SRC_YSTRIDE0 = stride;
MDP_SSPP_SRC_FORMAT = format_config;

// 3. Configure layer mixer input
MDP_LM_BLEND_CFG = alpha_blend_mode;
MDP_LM_BLEND_ALPHA = global_alpha;

// 4. Flush to make changes take effect at next vsync
MDP_CTL_FLUSH = (1 << pipe_id) | (1 << mixer_id);
```


__Visual Example__

```
Source Buffer (1920x1080 RGBA)
┌─────────────────────────────┐
│                             │
│  ┌──────────────┐          │ ← region = {100, 100, 800, 600}
│  │ Cropped Area │          │   (Only this part sent to display)
│  │              │          │
│  └──────────────┘          │
│                             │
└─────────────────────────────┘
      ↓ wfdBindSourceToPipeline
      ↓
MDP VIG0 Pipeline (Hardware Layer)
      ↓ Scaling/Blending/CSC
      ↓
Display (scaled to fit dst_rect)
┌─────────────────────────────┐
│                             │
│   ┌──────────────────┐      │
│   │   Scaled Image   │      │
│   │                  │      │
│   └──────────────────┘      │
│                             │
└─────────────────────────────┘
```

### End-to-End Example on QNX

```
// ========================================
// 1. APPLICATION LEVEL
// ========================================
screen_context_t screen_ctx;
screen_window_t win;
screen_buffer_t bufs[2];

// Initialize Screen
screen_create_context(&screen_ctx, SCREEN_APPLICATION_CONTEXT);
screen_create_window(&win, screen_ctx);

// Create double buffers
screen_set_window_property_iv(win, SCREEN_PROPERTY_BUFFER_COUNT, 2);
screen_create_window_buffers(win, 2);
screen_get_window_property_pv(win, SCREEN_PROPERTY_RENDER_BUFFERS, 
                            (void**)&bufs);

// Render frame to buffer
void *pixels;
screen_get_buffer_property_pv(bufs[0], SCREEN_PROPERTY_POINTER, 
                            (void**)&pixels);
// ... draw to pixels ...

// POST THE FRAME
screen_post_window(win, bufs[0], 1, NULL, 0);

// ========================================
// 2. SCREEN COMPOSITOR (QNX Process)
// ========================================
// Receives post notification via QNX message passing
compositor_handle_post(window, buffer) {
 // Get buffer physical address
 off64_t phys_addr;
 screen_get_buffer_property_llv(buffer,
                                SCREEN_PROPERTY_PHYSICAL_ADDRESS,
                                &phys_addr);
 
 // Get buffer format
 int format;
 screen_get_buffer_property_iv(buffer,
                               SCREEN_PROPERTY_FORMAT,
                               &format);
 
 // Assign to WFD pipeline based on window properties
 WFDPipeline pipeline = assign_pipeline(window);
 
 // Create WFD source from buffer
 WFDSource source = wfdCreateSourceFromBuffer(
     device, buffer, phys_addr, format);
 
 // ========================================
 // 3. BIND SOURCE TO PIPELINE
 // ========================================
 wfdBindSourceToPipeline(
     device,
     pipeline,
     source,
     WFD_TRANSITION_AT_VSYNC,
     NULL  // Use entire buffer
 );
 // At this point: STAGED but not visible yet!
 
 // ========================================
 // 4. COMMIT TO HARDWARE
 // ========================================
 wfdDeviceCommit(device, WFD_COMMIT_ENTIRE_DEVICE, 
                 WFD_INVALID_HANDLE);
}

// ========================================
// 5. QUALCOMM WFD DRIVER (libwfdqcom.so)
// ========================================
WFDErrorCode wfdDeviceCommit(WFDDevice device, ...) {
 DEVICE *dev = (DEVICE*)device;
 
 // Map MDP hardware registers
 uintptr_t mdp_base = mmap_device_io(MDP_REG_SIZE, MDP_BASE_ADDR);
 
 for (int i = 0; i < dev->num_pipelines; i++) {
     PIPELINE *pipe = &dev->pipelines[i];
     
     if (!pipe->needs_commit) continue;
     
     SOURCE *src = pipe->staged_source;
     
     // ========================================
     // 6. PROGRAM HARDWARE REGISTERS
     // ========================================
     
     // Configure source surface processor
     configure_sspp_registers(mdp_base, pipe->hw_pipe_id,
                             src->phys_addr,
                             src->format,
                             src->width, src->height,
                             pipe->src_rect);
     
     // Configure layer mixer
     configure_mixer_registers(mdp_base, pipe->mixer_id,
                              pipe->zorder,
                              pipe->alpha,
                              pipe->dst_rect);
     
     // Mark as committed
     pipe->needs_commit = false;
 }
 
 // ========================================
 // 7. TRIGGER HARDWARE UPDATE
 // ========================================
 
 // Flush configuration (applies at next vsync)
 uint32_t flush_mask = calculate_flush_mask(dev);
 out32(mdp_base + MDP_CTL_FLUSH, flush_mask);
 
 // Wait for vsync interrupt (optional)
 if (wait_for_vsync) {
     struct _pulse pulse;
     MsgReceivePulse(vsync_chid, &pulse, sizeof(pulse), NULL);
 }
 
 return WFD_ERROR_NONE;
}

// ========================================
// 8. HARDWARE (MDP)
// ========================================
// At next VSYNC interrupt:
// - MDP fetches pixels from physical memory
// - Applies scaling, color conversion
// - Blends with other layers in layer mixer
// - Sends to display interface (DSI/HDMI)

// ========================================
// 9. DISPLAY SHOWS FRAME
// ========================================
```

### Timing Diagram

```
Application Thread          Compositor Thread         Hardware
  |                            |                        |
  | screen_post_window()       |                        |
  |--------------------------->|                        |
  |                            | wfdBindSourceToPipeline|
  |                            |----------------------->|
  |                            | (staged, not visible)  |
  |                            |                        |
  | (can render next frame)    | wfdDeviceCommit()      |
  |                            |----------------------->|
  |                            |                        | Configure registers
  |                            |                        |
  | screen_post_window()       |                        |
  | (next frame)               |                        |
  |--------------------------->| (queued)               |
  |                            |                        |
  |                            |                        | VSYNC!
  |                            |                        |-----> Frame 1 visible
  |                            | wfdBindSourceToPipeline|
  |                            | (frame 2)              |
  |                            |----------------------->|
  |                            | wfdDeviceCommit()      |
  |                            |----------------------->|
  |                            |                        | VSYNC!
  |                            |                        |-----> Frame 2 visible
```


### Performance Optimizations

Zero-Copy Video Playback
```
// Video decoder writes directly to physically contiguous buffer
// MDP scans out directly - no CPU copies!

Physical Buffer ──────> MDP VIG Pipe ──────> Display
 (YUV)                 (YUV→RGB)          (RGB pixels)
                       Hardware CSC
```

UBWC (Universal Bandwidth Compression)
```
// Allocate UBWC-compressed buffer on QNX
int format = SCREEN_FORMAT_RGBA8888 | SCREEN_FORMAT_UBWC;
screen_set_buffer_property_iv(buf, SCREEN_PROPERTY_FORMAT, &format);

// MDP automatically decompresses during scanout
// Transparent to application
```

Hardware Composition
```
// Instead of CPU composition:
// ❌ CPU blends video + UI → single buffer → display

// Use hardware layers on QNX:
// ✅ Video buffer → VIG0 pipe ──┐
// ✅ UI buffer    → RGB0 pipe ──┼─→ Layer Mixer → Display
// ✅ Cursor       → DMA0 pipe ──┘
//
// No CPU involvement, much lower power!
```

__Asynchronous Commits__

```
// Non-blocking commit
wfdDeviceCommit(device, WFD_COMMIT_ENTIRE_DEVICE, WFD_INVALID_HANDLE);
// Returns immediately, flip happens at next vsync

// Application can start rendering next frame right away
screen_get_window_property_pv(win, SCREEN_PROPERTY_RENDER_BUFFERS, &bufs);
// Render to bufs[1] while bufs[0] is being displayed
```

__Dirty Rectangle Tracking__

```
// Only update changed regions
int dirty[] = {x, y, width, height};
screen_post_window(win, buf, 1, dirty, 0);

// Compositor only re-blends affected areas
// Saves composition bandwidth
```

__Format Optimization__

| Use Case | Recommended Format | Why |
|-----------|---------|---------|
| UI/Text | SCREEN_FORMAT_RGBA8888 | Alpha channel for blending |
| Photos | SCREEN_FORMAT_RGB888 | No alpha needed, saves memory|
| Video | SCREEN_FORMAT_NV12 | Native camera/decoder format |
| Low-power UI | SCREEN_FORMAT_RGB565  | Half the bandwidth |
---





                     
                     










































