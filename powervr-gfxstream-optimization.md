# Pixel 10 (PowerVR) & GfxStream Graphics Acceleration Guide

This guide details the graphics architecture, environment configurations, GameNative compatibility findings, and GfxStream hypervisor patches for **Google Pixel 10 (Tensor G5 - Imagination PowerVR D-Series DXT-48-1536 GPU)**.

---

## 1. Overview & Architecture Stack

$$\text{OpenGL App} \xrightarrow{\text{Mesa Zink}} \text{Vulkan 1.3} \xrightarrow{\text{GfxStream ICD}} \text{virtio-gpu (host\_visible)} \xrightarrow{\text{Host Driver}} \text{PowerVR GPU}$$

* **OpenGL Driver:** **Mesa Zink** (`MESA_LOADER_DRIVER_OVERRIDE=zink`), translating OpenGL 4.5 / ES 3.2 into Vulkan 1.3.
* **Vulkan ICD:** `GfxStream` (`/usr/share/vulkan/icd.d/gfxstream_vk_icd.json`).
* **Kernel DRM Driver:** `virtio-gpu-pci` with `+virgl`, `+resource_blob`, `+host_visible` shared memory windows.
* **Host Physical GPU:** Imagination PowerVR D-Series DXT-48-1536 MC1 (`IMAGINATION_PROPRIETARY`).

---

## 2. GameNative Pixel 10 / PowerVR Key Learnings

1. **Vulkan Adapter Enumeration Fix (`wrapper-gamenative`)**:
   * GameNative version **v0.9.0** introduced support for Pixel 10 / PowerVR via [PR #904](https://github.com/utkarshdalal/GameNative/pull/904).
   * Fixed Vulkan physical device / EGL adapter enumeration bugs in Wine/DXVK wrappers on PowerVR GPUs.
2. **ASurfaceRenderer Zero-Copy Presentation**:
   * Uses `VK_ANDROID_external_memory_android_hardware_buffer` to import frames into Android `AHardwareBuffer`s for direct scanout via `ASurfaceRenderer`.
3. **Automatic Driver Routing**:
   * GameNative automatically sets `DEFAULT_GRAPHICS_DRIVER = "Wrapper-gamenative"` for non-Adreno GPUs ([PR #1736](https://github.com/utkarshdalal/GameNative/pull/1736)).

---

## 3. Debian Container Environment Configuration

To ensure hardware acceleration stability across both Xwayland and Native Wayland applications, configure `/home/droid/labwc.env` and `/usr/local/bin/enable_gfxstream`:

```bash
# OpenGL & Vulkan Driver Routing
export MESA_LOADER_DRIVER_OVERRIDE=zink
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json
export MESA_VK_WSI_DEBUG=sw,linear

# Xwayland Acceleration & DRI2 Fallback (Prevents DRI3 host memory export crashes)
export XWAYLAND_NO_GLAMOR=1
export LIBGL_KOPPER_DRI2=1

# Native Wayland Toolkit Preferences
export GDK_BACKEND=wayland,x11
export QT_QPA_PLATFORM="wayland;xcb"
export ELECTRON_OZONE_PLATFORM_HINT=wayland
export SDL_VIDEODRIVER=wayland,x11
export MOZ_ENABLE_WAYLAND=1
```

---

## 4. GfxStream Host Backend Patch & Unrooted Deployment

### Applied Source Patches (`hardware/google/gfxstream/host/vulkan/VkCommonOperations.cpp`):

1. **DMA_BUF Handle Export Support**:
   ```cpp
   if (wantedGuestHandleType & VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT) {
       res.externalMemoryFeatures |= (VK_EXTERNAL_MEMORY_FEATURE_EXPORTABLE_BIT | 
                                      VK_EXTERNAL_MEMORY_FEATURE_IMPORTABLE_BIT);
   }
   ```
2. **Robustness2 Null Descriptor Support (for Mesa 26.1+)**:
   ```cpp
   VkPhysicalDeviceRobustness2FeaturesEXT robustness2Features = {
       .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_ROBUSTNESS_2_FEATURES_EXT,
       .nullDescriptor = VK_TRUE};
   ```

### Unrooted Host Deployment (`run_host_gfxstream.sh`):

Prepend the directory containing `libgfxstream_backend.so` to `LD_LIBRARY_PATH` prior to spawning `crosvm`:

```bash
export LD_LIBRARY_PATH=/home/droid/gfxstream_patched:$LD_LIBRARY_PATH
crosvm run --gpu=gfxstream,use_blob=true,use_vulkan=true ...
```

---

## 5. Verification Commands

```bash
# 1. Check Direct Rendering Status
glxinfo -B | grep -E "direct rendering|OpenGL vendor|Device"

# 2. Test Native Wayland Rendering
WAYLAND_DISPLAY=wayland-0 weston-simple-egl

# 3. Test Xwayland 60 FPS Pacing
DISPLAY=:0 glxgears
```
EOF
